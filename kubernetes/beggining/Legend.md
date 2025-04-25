# Ваша легенда по стеку Java / Kubernetes

_По вашему стеку Java еще я дополню вопросы на курсе (сейчас слёг со спиной)!_

Зарисуйте самостоятельно эту схему в Figma (и вы сразу поймёте всю картину и для начала вдохновитесь графической схемой Кубера в Google, найдите попроще и всё станет понятно) и воспроизведите в DeepSeek - по сути это финал курса. - Но на собеседовании это спросят точно по вопросу о том как вы взаимодействовали с Java и с Kubernetes?

## **Внутренняя структура банка:** отвечаем на два вопроса на собесе разом:
### Про опыт с 1. Java и с 2. Kubernetes, советую заучить и приступать уже сегодня (Антон спросил, а не рано нам писать в стеке: Java? - нет, Антон, не рано:)), по сути это ваш стек Java+K8S для ответов на собеседовании: 

- **У вас язык бэкенда** - это Java и у вас в Кубере развернуты адаптеры (30 штук - это джарники (jar) взаимодействующие между собой по методу API), развернуты на двух нодах (нода - это мастер-кластер (группа серверов) с подами внутри, а вторая нода 1:1 такая же - это резервирование первой).
- **Адаптеры** (сленг банка, а по сути джарник в контейнере) - это микросервисы вашего одного большого конвейера (написанного на Java).
- **Адаптеры были разные**: бизнесовые, калькулятор для кредита, документы и прочие отдающие API (вы не обязаны помнить все адаптеры).
- **Как работали с адаптерами**? Дергали за ручки API (делали API запрос) изнутри контейнера пода, если зависал клиент по базе данных PostgreSQL (не может пройти калькуляцию кредита - это отдельный адаптер калькулятора. Либо не может получить документы подписанные банком на кредит (скачать) - тоже отдельный адаптер - документ-адаптер. Что значит зависал клиент? Это значит, что клиент начал оформлять кредит и завис на этапе калькуляции на сайте ввода кредита (калькулятор-адаптер) - и дальше у него не кликается. Или клиент завис на скачивании документа который уже банк подписал чтобы ему кредит выдать (документ-адаптер).
- **Развернут кредитный конвейер** (большое Java приложение - смотреть выше) был в Kubernetes и по сути на один адаптер приходилось по 2-3 пода (под - это минимальная абстракция K8S над контейнером(-ми) внутри пода).
-**Мониторинг Кубера** был реализован с помощью cAdvisor (это агент который отдает метрики, а Prometheus их забирает) и kube-state-metrics.
- **kube-state-metrics** - это метрики состояния объектов K8s (Pods, Deployments, Nodes).
- **Далее соответственно доставка метрик в Grafana** на стандартный дашборд под названием Kubernetes.
- **Дашборд Kubernetes** скачивается в виде `JSON` с официального сайта Grafana Labs и выводится.

## Как работает K8S мониторинг (cAdvisor) и как мониторится Java конвейер (Actuator)?

### **Как всё устроено в Kubernetes мониторинге (простыми словами)**  

Представьте, что ваш Kubernetes-кластер — это **больница**, а мониторинг — это система наблюдения за пациентами (подами) и оборудованием (нодами).  

## **1. Кто за что отвечает?**  

### **(1) cAdvisor — «Датчики на пациентах»**  
- **Что делает:**  
  - Замеряет **пульс (CPU), давление (RAM), температуру (Disk I/O)** у каждого контейнера.  
  - Встроен в **kubelet** (как встроенные датчики в больничной койке).  

- **Где смотреть метрики:**  
  - Открыть `https://<нода>:10250/metrics/cadvisor` (но нужны права).  
  - Пример метрик:  
    ```bash
    container_cpu_usage_seconds_total{container="nginx"}  # Сколько CPU съел контейнер
    container_memory_usage_bytes{pod="frontend-123"}      # Сколько RAM заняло
    ```

### **(2) kube-state-metrics — «Медкарты пациентов»**  
- **Что делает:**  
  - Следит, **кто в коме (CrashLoopBackOff), кто выписан (Completed), кто только поступил (Pending)**.  
  - Не меряет пульс, а фиксирует **состояние** подов, деплойментов и нод.  

- **Где смотреть метрики:**  
  - Обычно работает в поде на порту `8080`.  
  - Пример метрик:  
    ```bash
    kube_pod_status_phase{phase="Running"}  # Сколько подов в работе
    kube_deployment_replicas_unavailable   # Сколько реплик упало
    ```

### **(3) Prometheus — «Главный врач, который собирает все данные»**  
- **Что делает:**  
  - Ходит по палатам (нодам), **спрашивает у cAdvisor** (как себя чувствуют контейнеры).  
  - Читает **медкарты** из kube-state-metrics (кто жив, кто мёртв).  
  - **Автоматически находит новые поды** (Service Discovery).  

- **Как настраивается:**  
  - В `prometheus.yml` пишутся правила:  
    ```yaml
    scrape_configs:
      - job_name: 'cadvisor'          # Сбор метрик контейнеров
        kubernetes_sd_configs:         # Автообнаружение
          - role: node
      - job_name: 'kube-state-metrics' # Сбор состояния подов
        kubernetes_sd_configs:
          - role: service
    ```

### **(4) Grafana — «Монитор в приёмном отделении»**  

- **Что делает:**  
  - Берёт данные от Prometheus и рисует **красивые графики**.  
  - Примеры дашбордов:  
    - **«Тяжелые пациенты»** — поды с высокой нагрузкой CPU.  
    - **«Кто в реанимации»** — поды в статусе `CrashLoopBackOff`.  

### **Полная схема мониторинга**

```mermaid
flowchart LR
    subgraph Kubernetes Cluster
        kubelet -->|"cAdvisor\n(метрики CPU/RAM/Диски)"| Prometheus
        kube-state-metrics -->|"статусы подов\n(Pods/Deployments)"| Prometheus
        JavaApp["Java App (Spring Boot)"] -->|"Actuator\n(/actuator/prometheus)"| Prometheus
    end

    Prometheus -->|"Service Discovery\n(автоматический поиск таргетов/целей)"| KubernetesAPI["Kubernetes API"]
    KubernetesAPI -->|"обновляет список\nподов/сервисов"| Prometheus
    Prometheus -->|"хранит данные"| Grafana
    Grafana -->|"дашборды"| Admin[["Админ"]]
```

### 🔍 [Как работает Service Discovery:](https://github.com/lamjob1993/linux-monitoring/blob/main/prometheus/beginning/8.%20%D0%A1%D0%B5%D1%80%D0%B2%D0%B8%D1%81-%D0%B4%D0%B8%D1%81%D0%BA%D0%B0%D0%B2%D0%B5%D1%80%D0%B8%20(Service%20Discovery).md)
1. **Prometheus спрашивает у Kubernetes API**:
   - _"Какие поды/сервисы/ноды у тебя есть?"_
   - Использует роли:
     - `role: pod` (для Java Actuator)
     - `role: service` (для kube-state-metrics)
     - `role: node` (для cAdvisor)

2. **Фильтрация через аннотации** (пример для Java Actuator):
   ```yaml
   # В Deployment пода Java-приложения:
   annotations:
     prometheus.io/scrape: "true"    # <- "Собирай мои метрики!"
     prometheus.io/port: "8080"      # Порт Actuator
     prometheus.io/path: "/actuator/prometheus"
   ```

3. **Prometheus обновляет таргеты/цели** каждые 30 сек (по умолчанию).

### 🛠️ Что куда подключается:
| Компонент            | Как обнаруживается                          | Пример метрик                     |
|----------------------|--------------------------------------------|-----------------------------------|
| **cAdvisor**         | Автообнаружение нод (`role: node`)         | `container_cpu_usage_seconds`     |
| **kube-state-metrics** | Сервис с фиксированным именем (`role: service`) | `kube_pod_status_phase`       |
| **Java Actuator**    | Поды с аннотацией `prometheus.io/scrape`   | `http_server_requests_seconds`    |

### ⚙️ Конфиг Prometheus (фрагмент):
```yaml
scrape_configs:
  # Для Java Actuator
  - job_name: 'java-apps'
    kubernetes_sd_configs:
      - role: pod  # Ищем поды
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"  # Берём только поды с аннотацией

  # Для cAdvisor
  - job_name: 'cadvisor'
    kubernetes_sd_configs:
      - role: node  # Ищем ноды

  # Для kube-state-metrics
  - job_name: 'kube-state-metrics'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: kube-state-metrics
```

### 🔄 Круговорот данных:
1. **Kubernetes API** сообщает Prometheus: _"Вот список всех подов/сервисов/нод"_.
2. **Prometheus** фильтрует таргеты/цели по аннотациям и ролям.
3. **cAdvisor/kube-state-metrics/Actuator** отдают метрики.
4. **Grafana** берёт данные (Data Source) из Prometheus и рисует графики.

Теперь админ видит **всё**:
- Нагрузку на контейнеры (cAdvisor) → _"Java-приложение жрёт 90% CPU!"_ / **похожая роль на экспортере Node Exporter**
- Статусы подов (kube-state-metrics) → _"Под calculator-adapter в статусе CrashLoopBackOff!"_ / **роль экспортера, который мониторит только состояние подов, перезапуск и т.д**
- Бизнес-метрики (Actuator) → _"API /orders обрабатывает 1000 RPS!"_ / **роль экспортера, который отдает метрики с Java-адаптеров**






























### **Как всё устроено в Kubernetes мониторинге (простыми словами)**  

Представьте, что ваш Kubernetes-кластер — это **больница**, а мониторинг — это система наблюдения за пациентами (подами) и оборудованием (нодами).  





---

## **1. Кто за что отвечает?**  

### **(1) cAdvisor — «Датчики на пациентах»**  
- **Что делает:**  
  - Замеряет **пульс (CPU), давление (RAM), температуру (Disk I/O)** у каждого контейнера.  
  - Встроен в **kubelet** (как встроенные датчики в больничной койке).  

- **Где смотреть метрики:**  
  - Открыть `https://<нода>:10250/metrics/cadvisor` (но нужны права).  
  - Пример метрик:  
    ```bash
    container_cpu_usage_seconds_total{container="nginx"}  # Сколько CPU съел контейнер
    container_memory_usage_bytes{pod="frontend-123"}      # Сколько RAM заняло
    ```

### **(2) kube-state-metrics — «Медкарты пациентов»**  
- **Что делает:**  
  - Следит, **кто в коме (CrashLoopBackOff), кто выписан (Completed), кто только поступил (Pending)**.  
  - Не меряет пульс, а фиксирует **состояние** подов, деплойментов и нод.  

- **Где смотреть метрики:**  
  - Обычно работает в поде на порту `8080`.  
  - Пример метрик:  
    ```bash
    kube_pod_status_phase{phase="Running"}  # Сколько подов в работе
    kube_deployment_replicas_unavailable   # Сколько реплик упало
    ```

### **(3) Prometheus — «Главный врач, который собирает все данные»**  
- **Что делает:**  
  - Ходит по палатам (нодам), **спрашивает у cAdvisor** (как себя чувствуют контейнеры).  
  - Читает **медкарты** из kube-state-metrics (кто жив, кто мёртв).  
  - **Автоматически находит новые поды** (Service Discovery).  

- **Как настраивается:**  
  - В `prometheus.yml` пишутся правила:  
    ```yaml
    scrape_configs:
      - job_name: 'cadvisor'          # Сбор метрик контейнеров
        kubernetes_sd_configs:         # Автообнаружение
          - role: node
      - job_name: 'kube-state-metrics' # Сбор состояния подов
        kubernetes_sd_configs:
          - role: service
    ```

### **(4) Grafana — «Монитор в приёмном отделении»**  
- **Что делает:**  
  - Берёт данные от Prometheus и рисует **красивые графики**.  
  - Примеры дашбордов:  
    - **«Тяжелые пациенты»** — поды с высокой нагрузкой CPU.  
    - **«Кто в реанимации»** — поды в статусе `CrashLoopBackOff`.  

---

## **2. Как всё связано?**  
```mermaid
flowchart LR
    kubelet -->|"cAdvisor (метрики CPU/RAM)"| Prometheus
    kube-state-metrics -->|"статусы подов"| Prometheus
    Prometheus -->|"хранит данные"| Grafana
    Grafana -->|"показывает графики"| Админ
```

---

## **3. Простая настройка (если лень читать)**  
1. **Ставим Prometheus + Grafana** (лучше через `kube-prometheus-stack`):  
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack
   ```
2. **cAdvisor и kube-state-metrics** уже включены в этот стек.  
3. **Готовые дашборды** появятся в Grafana:  
   - **Kubernetes / Compute Resources** (метрики cAdvisor).  
   - **Kubernetes / Kube-state-metrics** (статусы подов).  

---

## **4. Что важно запомнить?**  
- **cAdvisor** — отвечает за **CPU/RAM/Диск** контейнеров.  
- **kube-state-metrics** — отвечает за **статусы подов** (Running, Pending, Failed).  
- **Prometheus** — **собирает** данные от обоих.  
- **Grafana** — **показывает** это в графиках.  

Если что-то падает — сначала смотрите **логи подов**, потом **метрики cAdvisor**, потом **kube-state-metrics**. 🚑
