---
title: "TIKI scales data platform visualization với Apache Druid như thế nào"
date: 2021-09-19T23:10:44+07:00
publishdate: 2021-09-20T19:00:00+07:00
url: tiki-scales-data-platform-visualization-voi-apache-druid-nhu-the-nao
tags: ["apache-druid", "big-data", "data_engineering"]
resources:
- name: features
  src: images/druid-architect.webp
---

# Introduction

## Tại sao phải build hệ thống data phục vụ visualization ?

Vào đầu năm 2019, khi mà anh em bắt đầu mệt mỏi với backlogs dài hơn cầu Sài Gòn chỉ để viết SQL & build report Google Sheet thần thánh và Google Data Studio (để build dashboard report).

Lúc mà hiệu suất của anh em chậm lại đáng kể bởi:

- Việc build report bằng Google Sheet đa số là viết 1 câu SQL vài trăm dòng, lấy dữ liệu từ các bảng raw (dữ liệu thô) & từ vài chục GB đến trăm GB data (tốn hơn 110k để xử lý 1TB data), cực kì không hiệu quả.
- Để làm report trên Data Studio hiệu quả cần phải tối ưu khá nhiều thứ, ví dụ như thay vì dùng câu view thì phải build ra các bảng trước (materialized view). Nếu không cẩn thận thì tiền mất trong chốc lát là điều khó tránh khỏi.
- Chậm trể báo cáo ảnh hưởng đến việc không đủ dữ liệu đưa ra quyết định đúng đắn cho anh em business.

Về mặt dữ liệu còn gặp phải vấn đề:

- Cập nhật theo ngày, hoặc một số nhỏ hơn thì theo giờ: Việc không hổ trợ dữ liệu real-time sẽ ảnh hưởng nhiều đến vận hành, phân bổ về inventory hay là điều chỉnh về phân bổ các sản phẩm lúc những đợt sales lớn (ví dụ như 9.9, 10.10 ...).

Những điều đó thôi thúc anh em bắt đầu suy nghĩ tìm cách để tối ưu hóa hiệu suất cũng nhưng nâng cao hiệu quả sử dụng dữ liệu ở TIKI, với yêu cầu đặt ra về hệ thống này:

- Anh chị em Business (như team Marketing, ngành hàng, marketplace ...) có thể analyze một cách nhanh chóng & đưa ra các decisions từ những data đó. Đồng thời có thể tự tạo các dashboard chỉ bằng kéo thả, không cần phải mở tickets & đợi anh em data gửi data.
- Các bạn analytics, BI có thể tự viết cho mình một data pipeline, đưa lên hệ thống visualization một cách đơn giản nhất.
- Anh em data platform tâp trung vào build hạ tầng (Data Infrasturctures) & các công cụ (tools) để tổng hợp, tổ chức & xử lý data một cách hiệu quả, đơn giản. Phục vụ các teams & phòng ban khác ở TIKI thực hiện những phân tích nghiệp vụ cần thiết & đưa ra những quyết định đúng đắn mà không phải tốn quá nhiều công sức hay phải tìm hiểu quá sâu về mặt kỹ thuật (technicals).
- Dữ liêu phải được cập nhật real-time: ví dụ như khách hàng đặt đơn hàng thành công, thì đơn hàng đó ngay lập tức phải xuất hiện ngay lập tức trên dashboard.
- Đáp ứng những yêu cầu về kỹ thuật: ví dụ như lượng dữ liệu to dần (từ vài trăm GB lên cỡ TB) vẫn có thể đáp ứng được.
- Có thể mở ra một hướng mới để build report cho các backend ở TIKI

Để cho dễ hình dung thì các bạn có thể tham khảo từ ảnh dưới đây:

![sematix](./images/data_platform.gif)

*(Nguồn: Semantix - semantix.com.br)*

Hệ thống hiện tại sẽ giải quyết vấn đề Data Visualization & Data Sharing.

# Nghiên cứu & lựa chọn công nghệ

## Intro

Với những yêu cầu đã đặt ra, team bắt đầu tiến hành tìm kiếm & xâu dựng kiến trúc để giải quyết bài toán trên.

Trước tiên cần phải viết chi tiết hơn về mặt technical, hệ thống cần phải đáp ứng được:

Hệ thống này ổn định (reliable), phải nhanh (fast).

- Hệ thống này ổn định (reliable), phải nhanh (fast).
- Tích hợp được với hệ thống data warehouse hiện tại.
    - TIKI chọn BigQuery cho Data-Warehouse platform, vì vậy bắt buộc phải đồng bộ (sync) dữ liệu từ BigQuery
- Có thể backfill historical: ví dụ khi thêm column mới có thể run lại data quá khứ mà không tốn quá nhiều công sức.
- Hổ trợ data real-time, đặc biệt là từ Apache Kafka.
- Scalable: Dữ liệu growth từ GB lên TB hoặc vài chục TB vẫn phải đáp ứng được.
- Tiết kiệm: (+ auto scaling) có thể add thêm server khi cần thiết & giảm xuống vào thấp điểm.
- Không phải tốn quá nhiều nhân lực để quản lý & vận hành: (stable)

Sau một thời gian tìm hiểu, cài đặt & thử nghiệm các hệ thống khác nhau, trong đó có Apache Druid, Apache Pinot, ClickHouse. Thực sự lúc này so sách về ưu nhược điểm của các hệ thống trên khá ít. Và team cũng xác định cách tốt nhất vẫn là thử sai và thay đổi càng nhanh càng tốt để tìm ra được công nghệ phù hợp với yêu cầu của mình. Vì cơ bản, những công nghệ này hoàn toàn mới & team cũng chưa có nhiều kinh nghiệm trong việc vận hành.

Sau một thời gian thử nghiệm, team đã quyết định chọn Apache Druid là công nghệ chính cho hệ thống của mình.

## Apache Druid's coming

### What is Druid ?

> Apache Druid is a database that is most often used for powering use cases where real-time ingest, fast query performance, and high uptime are important. As such, Druid is commonly used for powering GUIs of analytical applications, or as a backend for highly-concurrent APIs that need fast aggregations. Druid works best with event-oriented data.  -  [https://druid.apache.org/](https://druid.apache.org/)

*Hiểu nôm na thì:*

> Druid giống như Google Sheet (hoặc như MySQL), có các cột & dòng tương ứng. Và nó giúp chúng ta có thể xử lý, tính toán hàng tỉ rows dữ liệu với tốc độ ánh sáng (1s tới vài giây).

Có một điều mình nhận thấy là document của druid viết rất kỹ, từ những cái đơn giản như Get started, đến các config, cách tuning cho productions ... Vì vậy, không cần phải đi hỏi  đâu xa, truy cập ngay hàng chính chủ [https://druid.apache.org/](https://druid.apache.org/) để tìm hiểu là ok nhất.

### Hiện tại TIKI đã sử dụng Druid cho những bài toán nào?

**Business intelligence / OLAP**

Druid là trái tim của toàn hệ thống OLAP của tiki, hiện tại tất cả những dữ liệu về Sales (in realtime), về users behaviors (xem phần dưới) đều sẵn sàng để phục vụ cho tất cả nhu cầu của anh em business.

**Clickstream analytics (web and mobile analytics)**

Đây có thể nói là lượng data khủng nhất từ trước đến giờ. Toàn bộ các actions của khách hàng trên Tiki Web/App đều được track & gửi về hệ thống tracking.

Team đã build cả data pipeline để tính toán lại gần như tất cả logic từ Google Analytics: như sessions, attributions models, new users ... Combine lại với các dữ liệu về Sales để tạo ra 1 data source duy nhất có thể đáp ứng được nhu cầu của business.

**Risk/fraud analysis:**

Khi có toàn bộ dữ liệu về sales, users behaviors thì việc tìm insights cho các hành vi fraud đơn giản hơn bao giờ hết.

**Application performance metrics**

Hiện tại để optimize về hiệu năng của web, app: anh em frontend & mobile đã gửi toàn bộ các tracking ví dụ như startup time, load pages ... Và team đã support để index vào Druid (in realtime). 

# Team đã vận hành Druid như thế nào?

![druid_architect.webp](./images/druid-architect.webp)

*Kiến trúc tổng thể để tích hợp Druid vào hệ thống hiện tại.*

## Đồng bộ dữ liệu từ BigQuery

Druid được thiết kế để có thể tận dụng được hệ thống data warehouse hiện tại, ví như như Hadoop, Hive, Spark, vì vậy gần rất nhiều cách đồng bộ dữ liệu (mình sẽ gọi là index). Tuy nhiên lại không có BigQuery trong danh sách :(.

Tìm kiếm một hồi là thấy được hổ trợ index files từ Google Cloud Storage - GCS (thông qua [Firehoses](https://druid.apache.org/docs/0.21.1/ingestion/native-batch.html#firehoses-deprecated) - hiện tại đang ngừng phát triển & nhường chổ cho công nghệ mới hơn là [Input Sources](https://druid.apache.org/docs/0.21.1/ingestion/native-batch.html#input-sources)) sử dụng `native batch indexer`.  Và mảng ghép còn lại chính là từ BigQuery qua GCS (đã được BigQuery hổ trợ sẵn). 

Điều kiện cần đã xong, tiếp theo mà make it done. Để gắn vào mảnh ghép data pipeline (TIKI sử dụng Airflow), team chỉ đơn giản là viết một airflow operator. 

Hello `BigQueryTableToOLAPOperator`, đối với các bạn sử dụng operator này, chỉ việc config vài dòng đơn giản:

```yaml
operator: BigQueryTableToOLAPOperator
table: tiktok_clone.girls_available
druid_destination_table: girls_available
```

Về technical bên dưới, `BigQueryTableToOLAPOperator` sẽ:

- Export bảng từ BigQuery sang Google Cloud Storage
- Build druid indexing spec (json)
- Submit spec vào Druid
- Liên tục kiểm tra status cho đến khi index xong (failed or sucess)
- Done.

## Scale & Optimization

### Phiên bản đầu tiên

Lúc này đối với TIKI thì Druid hoàn toàn mới, cũng không biết hỏi thăm ai về kinh nghiệm deploy như thế nào. Do vậy chỉ có cách là thử sai & rút ra kinh nghiệm.

Bắt đầu khiêm tốt 3 nodes, với kiến trúc của druid ở môi trường production cần phải có: ZooKeeper,  Coordinator, Overlord, Historical, Broker, Middlemanager

Quá nhiều components để quá thể cài đặt bằng cơm (ssh vào server & gõ yum install). Được anh em Infra tư vấn viết Ansible, thế là khăn gối đi học Ansible trước. Nhờ đầu óc sáng sủa, mất một tuần để viết được 1 em Ansible role. And voila, coi ngày & deploy thôi.

*Phiên bản production được hòa thiện và giới thiệu đến anh em business vào khoảng tháng 03/2019, chủ yếu phục các report về sales, thời điểm này chưa tới 50GB, anh em business kéo phát nào là ra số được phát đó. Êm lắm anh em <3.*

### Thất bại đau đớn

> *Cho đến 2 đợt event tiếp theo: 09/09/2019 & 10/10/2019, mỗi lần như vậy đều phải request thêm server để đủ đáp ứng được nhu cầu. Mỗi lần thêm như bật bắt buộc team phải thủ công đồng bộ lại Ansible*

Sau đó giới thiệu hệ thống olap cho các team khác, dữ liệu trong hệ thống không chỉ giới hạn ở sales mà đã bắt đầu mở rộng sang marketing (report về sessions, users, revenue cho các channel marketing), report về sellers, các dữ liệu về deal .... Từ vài chục GB đã lên con số hàng trăm GB.

Đợt đó là event TIKI Giựt Cô Hồn T08/2019, trăm anh em vào soi số trên hệ thống olap, và việc gì tới cũng tới. Toang hết cả làng anh em ạ.

Sau đó mới phải xin thêm 3 con VM nữa, lại cặm cụi Ansible. Sau gần 3h thì hệ thống bắt đầu êm lai. 500 anh em tiếp tục soi số.  

Vì vậy lúc này, team nghiên cứu con đường khác để có thể scale nhanh hơn nữa.

### Path to Kubernetes

Vào lúc nghiên cứu cách vận hàng druid trên k8s thì team cũng đã có kha khá kinh nghiệm vận hành Apache Airflow & Apache Spark trên k8s. (Tham khảo ở đây)

[Path to airflow 2](https://www.hienph.dev/posts/apache-airflow/)

Bắt đầu research thử xem có nhiều documents về cách deploy hay không, chỉ tìm được vài bài thôi anh em ạ. Trong đó có 1 em là helm chart. Cơ mà lại outdate cộng với 1 nhược điểm nữa là phải xin quyền để xài được helm, cực đủ bề.

Hết cách, ngồi tay chân viết kubernetes manifest thôi.

Trước khi thiết kế, có 1 vài yêu cầu mình cần nghĩ tới:

- Làm sao có thể chia thành nhiều tier (group) được: ví dụ data về sales sẽ được đặt vào 1 nhóm server ổn định, hoặc nhiều resources hơn. Data còn lại cho vào 1 cụm khác.
- Rẻ nhất có thể: Phải run được trên Preemtible Node (Preemtible tiết kiệm 70% so với những node thông thường & điểm đặc trưng mà thời gian tối đa từ lúc được tạo là 24h, có thể bị down bất kì thời điểm nào).
- Batch indexer có thể auto scale khi có nhu cầu về backfill dữ liệu quá khứ hoặc là những giờ cao điểm. Còn lại scale down để tiết kiệm chi phí.

Với những yêu cần như trên thì mình tổ chức như sau:

- ZooKeeper ( 3 Replicas, StatefulSet)
- Coordinator (1 Replica): chạy trên stable node pool
- Overlord (1 replica): : chạy trên stable node pool
- Middleware: chia thành 2 category (đều chay trên preemtible) (dựa vào config `druid.worker.category`)
    - Batch Category: Đảm nhiệm index data từ BigQuery
    - Real-Time category: Sync data từ Kafka
- Broker: (2 replicas để đảm bảo HA) (chạy trên preemtible)
- Historical: mình chia thành 2 nhóm (tier)
    - core: chứa data về sales
    - default: tất cả data còn lại.

Việc chia tier cho historicals & middlewares đặt vấn đề làm sao mình có thể tái sử dụng được manifest, vì về bản chất các tier đến khoảng 90% là giống nhau. Ngoài helm ra còn có  [kustomize](https://github.com/kubernetes-sigs/kustomize) có thể giúp mình được.

Thiết kế đã xong, tiến hay bắt tay viết & kiểm tra manifests. Và tiếp theo là quá trình migrate toàn bộ hệ thống lên Kubernetes.

### Migrate từ VM lên Kubernetes.

Để hạn chế downtime đến mức tối thiểu như sau, team tiến hành shutdown các components khác chỉ đê lại 2 phần chính: Broker & Historical (Trước đó đã xác nhận broker vẫn có thể query khi tắt coordinator & các components khác).

Tiến hành apply manifest với replica = 0. Start lần lượt các services: ZooKeeper → Coordinator → Historical → Broker.

Đợi Historical Pull hết data từ deep storage.

Sau đó là switch endpoint to cụm druid mới.

Cuối cùng là hoàn tất các component còn lại: Overlord, Batch Indexer & Realtime Indexer.

Toàn bộ quá trình diễn ra chưa đến 30 phút.

### Monitor & Optimize

**Bật cached ở historical**

Để tăng tốc query thì mình sử dụng memcached ( 3 nodes + facebook mcrouter để scale & shard memcached) ở Historical. (Bật cache ở broker có thể dẫn đến lệch data đó nhé, cẩn thận)

**Bật memory tối đa có thể cho druid**

Druid sử dụng page cache để optimize việc read segment từ disk. Vì vậy, càng nhiều RAM trống, càng tối ưu cho page cache.

**Monitor**

Hiện tại mình expose metrics thông qua statsd-exporter & sử dụng prometheus & grafana để index & monitor cũng như alerting.

**Tuning Config**

Để đảm bảo hệ thống luôn đáp ứng được hiệu năng khi lượng dữ liệu tăng lên, team định kỳ sẽ tuning lại cái config dựa theo các guidelines của Druid. Và hiện tại một vài config team nhận thấy ảnh hưởng nhiều đến hiệu năng nhất như `size of buffers`, `processing threads`, `memory allocated to query caches` .

# Kết luận

Sau quá trình dài đằng đẵng để tuning & optimize cho usecase của TIKI, druid đã chứng tỏ được khả năng scale để đáp ứng được nhu cầu của 500 em.

Một điều mình rất thích ở druid là toàn bộ data được lưu trữ ở deep storage (hiện tại đang sử dụng gcs), nên việc migrate hệ thống cực kỳ đơn giản, không lo lắng mất mát dữ liệu.

Ở thời điểm hiện tại có thể nói Druid đã cự kỳ stable, và nhờ vào hệ thống self-healing, việc running toàn bộ historical & broker trên Preemtible VM không ảnh hưởng quá nhiều đến data & performance. Tuy nhiên, vẫn còn nhiều thứ phải optimize, đặc biệt là disk usages, hiện tại billing disk đã gấp 1.5 lần CPU + Memory cộng lại.

Hiện tại Hệ thống đang ingest realtime hơn 20 k/s events, vào đợt campaign có lúc hệ thống ghi nhận được gần **100 k/s events** & query gần **13TB (26TB replicated)** data với hơn **65 tỉ rows** để get được nhiều chi tiết nhất về customers experiences, sales, inventory ...
