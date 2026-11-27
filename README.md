# aws-tf-springboot-manifests

[aws-terraform-springboot-infra](https://github.com/yeopjoonhyun/aws-terraform-springboot-infra)의 Spring Boot 앱을 EKS에 배포하는 쿠버네티스 매니페스트입니다. Argo CD가 이 레포의 `overlays/prod`를 지켜보고 클러스터를 이 내용에 맞춥니다.

## 배포 흐름

1. 인프라 레포의 main에 커밋이 들어오면 GitHub Actions가 ARM64 이미지를 ECR에 올린다.
2. 같은 작업이 이 레포의 `base/kustomization.yaml`에서 `newTag`만 새 커밋 해시로 바꿔 커밋한다. 이 레포에만 쓸 수 있는 배포 키를 쓴다.
3. Argo CD가 새 커밋을 보고 Deployment를 바꾼다. 쿠버네티스가 파드를 하나씩 교체한다.
4. 되돌리기는 이 레포에서 태그 커밋을 `git revert`한다.

클러스터에 직접 `kubectl apply`하지 않는다. 손으로 바꾼 내용은 Argo CD가 Git 상태로 되돌린다(selfHeal).

## 구성

| 파일 | 역할 |
|---|---|
| `base/deployment.yaml` | 앱 파드 2~4개. 8080은 요청, 8081은 지표와 관리용 |
| `base/service.yaml`, `base/ingress.yaml` | ALB Controller가 인터넷용 ALB를 만든다. 8080만 연결 |
| `base/hpa.yaml` | CPU 60% 기준으로 파드 2~4개 |
| `base/servicemonitor.yaml` | Prometheus가 8081의 `/actuator/prometheus`를 15초마다 수집 |
| `base/prometheusrule.yaml` | 골든 시그널 경보 4개. Alertmanager가 Slack으로 보낸다 |
| `overlays/prod/` | 네임스페이스 `saa-notes` |

## 레포에 없는 것

- **ECR 주소.** 주소에 AWS 계정 번호가 들어 있어 공개 레포에 두지 않는다. 매니페스트에는 `image: app`만 두고, 클러스터에만 있는 Argo CD Application이 이름을 ECR 주소로 덮어쓴다. 이때 Argo CD는 같은 층(overlay)의 태그를 지우므로 태그는 한 층 아래(base)에 둔다.
- **DB 접속 정보.** 인프라 레포의 `k8s/deploy.sh`가 Secrets Manager에서 꺼내 쿠버네티스 Secret `app-db`로 넣는다.
