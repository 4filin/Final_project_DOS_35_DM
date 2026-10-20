# Final Project: CI/CD + IaC для Go-приложения в AWS

Дипломный проект: полный цикл эксплуатации веб-приложения — от коммита до
работающего стенда с мониторингом и централизованным логированием.
База — форк [golang-helloworld](https://github.com/hackersandslackers/golang-helloworld),
модернизирован: Go 1.25, юнит-тесты, Prometheus-метрики, пробы `/healthz` `/readyz`,
graceful shutdown, структурное логирование запросов.

## Архитектура

```
git push (master) ──► GitHub Actions: lint → build → test → образ(:SHA) → deploy
                                                          │
                                                          ▼
                        EKS (managed control plane; 2 worker-ноды × 2 пода)
                              │ NodePort :30080
                              ▼
                    nginx reverse-proxy (EC2) :80  ◄── Пользователи

Мониторинг:  Prometheus + Grafana (отдельная EC2; scrape приложения через nginx)
Логирование: Filebeat (DaemonSet в EKS) → Elasticsearch + Kibana (выделенная EC2, us-west-2)
Уведомления: Telegram Bot (результаты каждого пайплайна)
```

## Стек

| Слой | Инструменты |
|---|---|
| Язык | Go 1.25, gorilla/mux |
| CI/CD | GitHub Actions (3 job'а; деплой только из `master`) |
| Контейнеры | Docker (мультистейдж Dockerfile), Amazon ECR |
| IaC | Terraform (state в S3, 2 региона) |
| Конфигурация | Ansible (динамический inventory по тегам AWS + статический для ELK-ноды) |
| Оркестрация | AWS EKS (k8s 1.35, managed node group 1/2/3 × t3.small) |
| Балансировка | nginx reverse-proxy  |
| Мониторинг | Prometheus, Grafana (дашборд node-exporter 1860) |
| Логирование | Elastic: Filebeat 8.14 → Elasticsearch 8.14 → Kibana 8.14 |
| Уведомления | Telegram |

## Развёртывание с нуля

```bash
# 1. Инфраструктура: EC2, ECR, EKS (us-west-1) + ELK-нода со своей VPC (us-west-2)
cd terraform && terraform apply

# 2. Конфигурация серверов: Docker, nginx, мониторинг, ELK
cd .. && ansible-playbook -i ansible/aws_ec2.yml -i ansible/elk_static.yml ansible/playbook.yml

# 3. Доступ к кластеру
aws eks update-kubeconfig --region us-west-1 --name final-project

# 4. Деплой приложения (CI соберёт образ и применит k8s-манифесты)
git push origin master
```

Bootstrap (один раз в жизни): S3-бакет под Terraform state —
`aws s3 mb s3://dos35-md-final-project-study --region us-west-1`.
GitHub Secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SSH_PRIVATE_KEY`,
`TELEGRAM_TOKEN`, `TELEGRAM_TO`.

⚠️ После пересоздания инфраструктуры (`destroy`/`apply`) IP меняются — обновить в 4 файлах:
`ansible/playbook.yml` (upstream nginx), `ansible/elk_static.yml`, `k8s/filebeat.yaml`
(output ES), `monitoring/prometheus.yml` (target app-eks).

## CI/CD пайплайн

| Job | Триггер | Действия |
|---|---|---|
| `build-and-test` | любой push / PR | golangci-lint (v2.5.0) → `go build` → `go test -cover` → `docker build` |
| `deploy` | только `master` | push образа (:SHA + :latest) в ECR → `update-kubeconfig` → `kubectl apply -f k8s/` → `set image` по SHA → rollout status |
| `notify` | всегда | Telegram: ветка, коммит, результаты job'ов |

## Проверка работы

| Что | Где |
|---|---|
| Приложение | `http://<app_server_ip>/` |
| Prometheus targets | `http://<app_server_ip>:9090/targets` |
| Grafana (admin/admin) | `http://<app_server_ip>:3000` |
| Kibana (Discover) | `http://<elk_public_ip>:5601` |

## Структура репозитория

```
├── main.go, main_test.go, Dockerfile, go.mod   # приложение
├── .github/workflows/ci-cd.yaml                # CI/CD
├── terraform/                                  # provider/backend, ecr, ec2, eks, elk
├── ansible/                                    # playbook (3 play'я), inventories, group_vars
├── k8s/                                        # deployment, service (NodePort), filebeat
├── monitoring/                                 # prometheus.yml, compose-файлы стеков
└── part1..part6                                # документация по этапам
```

## Документация

- [step1-apps_docker.md](step1-apps_docker.md) — репозиторий, Docker, тесты
- [step2-cicd-pipeline.md](step2-cicd-pipeline.md) — CI/CD, разделение по веткам
- [step3-infrastructure.md](step3-infrastructure.md) — Terraform + Ansible
- [step4-monitoring.md](step4-monitoring.md)— Prometheus + Grafana
- [step5-kubernetes.md](step5-kubernetes.md) — EKS, манифесты, балансировка
- [step6-logging](step6-logging) — Filebeat + Elasticsearch + Kibana



