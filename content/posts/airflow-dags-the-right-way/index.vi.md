---
title: "Airflow Dags The Right Way"
date: 2021-08-31T15:09:24+07:00
publishdate: 2021-08-31
tags: ['airflow', 'data_engineering']
resources:
- name: features
  src: images/dags-config.png
---

# Bối cảnh

Bài viết này là cách mình thiết kế & tổ chức config cho airflow (trước thềm đú trend lên cloud).

Sau khi quá mệt mỏi với việc copy dags từ nhà lên đám mây, copy tới đâu gãy tới đó. Thế là nông dân quyết tâm thiết kế một cách viết riêng để có thể viết dag một cách ổn định nhất nhất, kể cả khi core của airflow thay đổi (ví dụ import path thay đổi, params thay đổi ...)

Full link bài viết ở đây nhé.

[Path to airflow 2](https://www.hienph.dev/posts/apache-airflow/)

Những vấn đề đối với cách viết dags hiện tại:

- Những chuổi ngày copy - paste lăp đi lặp lại, vô vị (nghe giống DRY - Don't Repeat Your Self Principle không). Nếu không được tổ chức đúng cách, code reviews có thể dẫn đến con đường đập đi xây lại một ngày không xa (duplicated and unmaintainable code).
- Non - tech users (các bạn analytics, BA ...) phải tốn thời gian học một ít python, import module như thế nào cho đúng. Điều này dẫn đến một lúc nào đó các chú culi 4.0 (aka data engineer) phải ngồi viết dùm dags cho các em xinh đẹp =)) (Cho chừa tội mê gái)
- We can do better!

# Từng bước nghiên cứu & thiết kế hệ thống dag config

(Lúc này là tháng 09/2019) Mình bắt đầu thử với các keyword: dag config, dag factory, dag yaml

Research 1 hồi thì tìm được [https://github.com/ajbosco/dag-factory](https://github.com/ajbosco/dag-factory), tương lai đây rồi.

```yaml
example_dag1:
  default_args:
    owner: 'example_owner'
  schedule_interval: '0 3 * * *'
  tasks:
    task_1:
      operator: airflow.operators.bash_operator.BashOperator
      bash_command: 'echo 1'
    task_2:
      operator: airflow.operators.bash_operator.BashOperator
      bash_command: 'echo 2'
      dependencies: [task_1]
```

Với dag-factory thì có 1 số vấn đề mình cần giải quyết:

- users vẫn phải input full cái import path cho operator → Cần pải tạo alias name
    - Ví dụ `airflow.operators.bash_operator.BashOperator` → `BashOperator`
- Chỉ cần input những thông tin quan trong.
    - vd với operator `airflow.contrib.operators.bigquery_operator.BigQueryOperator` chỉ cần truyền vào: `sql` , `destination_dataset_table`. Không cần phải truyền thêm `gcp_conn_id`, cái option như `create_disposition`, `write_disposition`
- Tự động phân quyền dags & set connection_id tương ứng cho mỗi team.
    - vd team 1 thì dùng `bigquery_conn_id=team1`, dù users có truyền connection_id thì vẫn phải override ở code.
- Cần phải force một số conventions:
    - Mỗi team sẽ có 1 prefix riêng, tiện cho việc phân biệt.
    - Tên file = tên dag → Dễ debug khi có biến.

⇒ Phải đổi một xíu cái lib dag factory này.

## **Let's start !**

Với những tinh hoa được học từ thanh niên cứng (SOLID), Open for Extension, Closed For Modification. Giờ không đuợc thay đổi code của `dag-factory` mà mình sẽ extend nó.

Tức là thay vì vào edit code để support gắn `conn_id`, `operator alias` thì mình sẽ tạo thêm 1 layer phía trên và tiến hành convert nó đúng với format mà `dag-factory` cần.

Bắt đầu với thiết kế alias cho `operators`

```python
class OperatorAlias:
    # alias name: BigQueryOperator
    name: str

    # full module path: airflow.providers.google.cloud.operators.bigquery.BigQueryExecuteQueryOperator
    module: str

    # json schema, use for validate the user inputs.
    # {"type": "object", "properties": {"sql": {"type": "string", "minLength": 2 }}, "required": ["sql"]}
    schema: dict

    # set the default params like connections ...
    # {"bigquery_conn_id": "gcp_girls", "allow_large_results": true, ...}
    default_params dict
```

Với thiết kế 1 alias như trên, bây giờ thay vì phải viết một file python như thế này

```python

from airflow import DAG
from airflow.providers.google.cloud.operators.bigquery import BigQueryExecuteQueryOperator
from .. import BigQueryTableToOLAPOperator

default_args = {
    'owner': 'my@names.com',
}
with DAG(
    'tutorial',
    default_args=default_args,
    schedule_interval='0 3 * * *',
    tags=['dwh','etl'],
) as dag:
    t1 = BigQueryExecuteQueryOperator(
        task_id='ext_girls',
        sql='SELECT * FROM girls WHERE age >= 18 and age <= 30',
        destination_dataset_table='tiktok_clone.girls_available',
        gcp_conn_id='gcp_girls',
        create_disposition="CREATE_IF_NEEDED",
        write_disposition="WRITE_TRUNCATE",
        allow_large_results=True,
    )
    t2 = BigQueryTableToOLAPOperator(
        task_id='sync_to_olap',
        table='tiktok_clone.girls_available',
        gcp_conn_id='gcp_girls',
        druid_conn_id='k8s_druid',
        date_column='date',
    )
		t1 >> t2
```

Chỉ cần viết 1 file yaml.

```yaml
# ext_available_girls.yaml
default_args:
  owner: 'my@names.com'
*schedule_interval*: '0 3 * * *'
tags:
  - dwh
  - etl
tasks:
  ext_girls:
    operator: BigQueryOperator
    sql: 'SELECT * FROM girls WHERE age >= 18 and age <= 30'
    destination_dataset_table: tiktok_clone.girls_available
  sync_to_olap:
    operator: BigQueryTableToOLAPOperator
    table: tiktok_clone.girls_available
    druid_destination_table: girls_available
    dependencies:
      - ext_girls
```

Xịn rồi, coi ngày & deploy thôi anh em ơi.

## Phân Quyền

PoC (Proof of Concept) cơ bản đã hoạt động được, tiếp theo cần phải giải quyết vấn đề tự động phân quyền & set các connections for mỗi team.

Mỗi team sẽ có 1 role riêng, các DAG sẽ được gắn quyền read trên role này.

Ý tưởng ban đần là mỗi team sẽ có role riêng, các connection_id, & alert connection_id riêng luôn.

```yaml
teams:
  - name: finances
    role_id: 6
    alert:
      kind: telegram
      conn_id: fin_alert
    conns:
      - conn_id: gcp_team_2
        replace_fields:
          - bigquery_conn_id
          - gcp_conn_id
          - google_cloud_storage_conn_id
          - google_cloud_conn_id
```

Với config như trên thì biết đọc yaml ở folder nào, thế này chế ra phải bắt buộc các file yaml phải được đặt vào thư mục với `name` tương ứng. Ví dụ (`dags/finances`)

Đến đây chỉ việc viết 1 job sương sương duyệt folder, parse yaml và:

- Gắn `failed_callback` tương với alert connection_id.
- Nếu match ``replace_fields`` thì tiến hành thay thế luôn.
- Về roles: Sau khi nghiên cứu thì mình phát hiện Airflow sử dụng flask-appbuilders, dẫn đến chỉ cần viết 1 câu SQL nhỏ nhỏ để insert quyền `read` vào bảng `ab_permission_view_role` với `dag_id` là đủ xài (Hack nhé, cẩn thận sập =]])

# Kết luận

- Không cần biết python vẫn viết dag được.
- `Declarative` & `Abstraction`: Users không cần phải biết quá chi tiết về mỗi operator có những gì, chỉ cần điền những field đủ để run (tất nhiên vẫn cần phải đủ flexible để có thể tùy biến khi cần thiết)
- Tự động gắn alert khi dag failed. (Mà đối với python phải import tay vào từng DAG).
- Dễ cho việc upgrade airflow: Bây giờ việc upgrade airflow không còn là ám ảnh.
    - Nếu airflow đổi import path ⇒ mình chỉ cần tạo đổi module path trong bảng operators là xong.
    - Nếu airflow đổi field name, mình tạo 1 operator adapter và trỏ module path tới operator mình vừa tạo.
    - Life's so easy.
- Tự động phân quyền:
    - Thực tế ở TIKI có khá nhiều team, & mỗi team muốn dag nhà ai nấy ở.
    - Vì vậy mình đã chia mỗi team 1 có 1 folder riêng trong git, hoặc thậm chí là 1 git repo riêng luôn,  có role riêng. Mỗi khi gen dag thành công, thì cũng sẽ auto update role tương ứng cho dag đó.
- Những thiết kế này chỉ là bước đầu, còn rất nhiều room để cải thiện thêm, 1 case rất điển hình như: Kéo thả dags nì: Thay vì phải ngồi viết yaml, cực nhọc học git, chỉ việc lên UI kéo thả các thứ & Tạo ngay 1 dags cho mình.

Với những tóm tắm ở trên hy vọng các bạn các thể chế ra được một phiên bản airflow khác, miễn sao mình bản thân sử dụng & happy với nó là được.

# Resources
Những thiết kế này mình đã hoàn thành vào 2019, nhưng mà idea của nó mình vừa gặp lại 2 ở 2 bài viết khá hay.
- [Data Engineers Shouldn't Write Airflow Dags](https://towardsdatascience.com/data-engineers-shouldnt-write-airflow-dags-b885d57737ce)
- [Data Engineers Shouldn't Write Airflow Dags - Part 2](https://towardsdatascience.com/data-engineers-shouldnt-write-airflow-dags-part-2-8dee642493fb)
