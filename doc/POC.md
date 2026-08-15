# PoC: розгортання GitOps-системи ArgoCD на k3d

**Проєкт:** AsciiArtify
**Етап:** Proof of Concept
**Кластер:** k3d (обрано на етапі [Concept](Concept.md))
**Версія ArgoCD:** v3.5.1
**Автор:** Oleksandr Kaminskyi

---

## 1. Мета

Перевірити технічну можливість розгортання GitOps-системи на затвердженому етапі Concept варіанті Kubernetes та надати команді розробки доступ до графічного інтерфейсу ArgoCD.

Результат етапу - працездатна інсталяція ArgoCD, готова до реалізації MVP.

---

## 2. Передумови

| Компонент | Версія | Перевірка |
|---|---|---|
| Docker | 20.10+ | `docker version` |
| k3d | 5.9+ | `k3d version` |
| kubectl | 1.32+ | `kubectl version --client` |
| argocd CLI | 3.x | `argocd version --client` |

Встановлення argocd CLI:

```bash
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```

Якщо кластер із попереднього етапу ще існує - видалити:

```bash
k3d cluster list
k3d cluster delete asciiartify
```

---

## 3. Крок 1. Розгортання кластера

Порт 80 вбудованого load balancer'а k3d прокидається на `localhost:8080` - саме через нього працюватиме доступ до UI.

```bash
k3d cluster create asciiartify \
  --image rancher/k3s:v1.32.2-k3s1 \
  --agents 2 \
  -p "8080:80@loadbalancer" \
  --wait

kubectl get nodes
```

```
NAME                       STATUS   ROLES                  AGE   VERSION
k3d-asciiartify-agent-0    Ready    <none>                 14s   v1.32.2+k3s1
k3d-asciiartify-agent-1    Ready    <none>                 17s   v1.32.2+k3s1
k3d-asciiartify-server-0   Ready    control-plane,master   27s   v1.32.2+k3s1
```

---

## 4. Крок 2. Встановлення ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available --timeout=300s \
  deployment --all -n argocd

kubectl get pods -n argocd
```

> Прапорці `--server-side --force-conflicts` рекомендовані офіційною документацією через обмеження на розмір CRD.

Мають запуститися: `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, `argocd-applicationset-controller`, `argocd-dex-server`, `argocd-notifications-controller`, `argocd-redis`.

---

## 5. Крок 3. Налаштування доступу через Traefik

За замовчуванням `argocd-server` сам термінує TLS. Якщо перед ним поставити Traefik, який теж термінує TLS, виникає нескінченний цикл редиректів HTTP → HTTPS. Тому сервер переводиться в insecure-режим, а TLS залишається на Traefik.

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout status deployment argocd-server -n argocd
```

Ingress (Traefik у k3s працює з коробки, встановлювати нічого не потрібно):

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
spec:
  rules:
    - host: argocd.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: http
EOF

kubectl get ingress -n argocd
```

```
NAME            CLASS     HOSTS              ADDRESS                            PORTS   AGE
argocd-server   traefik   argocd.localhost   172.18.0.3,172.18.0.4,172.18.0.5   80      0s
```

> **Windows / WSL:** якщо `argocd.localhost` не резолвиться, додайте рядок `127.0.0.1 argocd.localhost` у `C:\Windows\System32\drivers\etc\hosts`.

---

## 6. Крок 4. Перший вхід

### 6.1 Веб-інтерфейс

Відкрити **http://argocd.localhost:8080** - має з'явитися форма входу ArgoCD.

Початковий пароль адміністратора зберігається в секреті:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### 6.2 CLI

> **Важливо.** Оскільки сервер працює в insecure-режимі (plain HTTP, TLS на Traefik), звичні прапорці CLI не спрацюють:
>
> - `--insecure` означає «не перевіряти TLS-сертифікат», але з'єднання все одно йде по TLS → сервер відповідає `404 Not Found`
> - argocd CLI за замовчуванням використовує gRPC, який звичайний Ingress не пропускає без окремого роутингу
>
> Правильна комбінація - `--plaintext --grpc-web`:

```bash
argocd login argocd.localhost:8080 \
  --username admin \
  --plaintext --grpc-web
```

Після успішного входу контекст зберігається в `~/.config/argocd/config` і використовується наступними командами автоматично.

> **Нюанс.** Збережений контекст має пріоритет над прапорцями при повторному вході: якщо перший `login` пройшов по TLS, наступні спроби з `--plaintext` продовжать видавати попередження про сертифікат. Скидається через `argocd logout argocd.localhost:8080`.

Обов'язкова зміна пароля адміністратора:

```bash
argocd account update-password
```

Початковий секрет після цього видаляється - він більше не потрібен і не має лежати в кластері:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### 6.3 Альтернативний доступ (діагностика)

Якщо Ingress не працює, доступ можна отримати напряму, минаючи Traefik. Порт 8081 - тому що 8080 на хості вже зайнятий load balancer'ом k3d:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

З'єднання йде на порт 443 сервісу із самопідписаним сертифікатом, тому прапорці інші:

```bash
argocd login localhost:8081 --username admin --insecure
# UI: https://localhost:8081
```

Якщо через `port-forward` усе працює, а через Ingress ні - проблема в маршрутизації Traefik, а не в самому ArgoCD.

---

## 7. Крок 5. Доступ команди

Обліковий запис `admin` є суперкористувачем, на якого не поширюються RBAC-політики, і команді він не роздається. Для двох розробників створюються окремі акаунти з обмеженими правами.

### 7.1 Створення облікових записів

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '
data:
  accounts.dev1: apiKey, login
  accounts.dev2: apiKey, login
'
kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout status deployment argocd-server -n argocd
```

Перевірка:

```bash
argocd account list
```

```
NAME   ENABLED  CAPABILITIES
admin  true     login
dev1   true     apiKey, login
dev2   true     apiKey, login
```

### 7.2 Права доступу (RBAC)

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p '
data:
  policy.default: ""
  policy.csv: |
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, logs, get, */*, allow
    p, role:developer, projects, get, *, allow
    g, dev1, role:developer
    g, dev2, role:developer
'
```

Розробник бачить застосунки, може їх синхронізувати та читати логи, але не може створювати чи видаляти застосунки, змінювати проєкти або керувати обліковими записами. `policy.default: ""` означає заборону всього, що не дозволено явно.

### 7.3 Встановлення паролів

Паролі задаються адміністратором і передаються розробникам поза репозиторієм - у git вони не потрапляють у жодному вигляді.

```bash
argocd account update-password \
  --account dev1 \
  --current-password '<ADMIN_PASSWORD>' \
  --new-password '<DEV1_PASSWORD>'
```

> `--current-password` - це пароль **адміністратора**, під яким виконується команда, а не попередній пароль dev1.
> Значення беруться в одинарні лапки: символи `!`, `#`, `$` інтерпретуються оболонкою.

Щоб пароль не потрапив в історію shell:

```bash
read -s ADMIN_PW && read -s DEV1_PW
argocd account update-password --account dev1 \
  --current-password "$ADMIN_PW" --new-password "$DEV1_PW"
unset ADMIN_PW DEV1_PW
```

### 7.4 Перевірка обмежень

Авторизація перевіряється на стороні сервера. Вхід під обліковим записом розробника:

```bash
argocd login argocd.localhost:8080 --username dev1 --plaintext --grpc-web
```

```
'dev1:login' logged in successfully
Context 'argocd.localhost:8080' updated
```

```bash
argocd account can-i sync applications 'default/guestbook'      # yes
argocd account can-i delete applications 'default/guestbook'    # no
argocd account can-i create applications 'default/*'            # no
```

```
yes
no
no
```

> **Важливо для розуміння.** Веб-інтерфейс відображає кнопки `SYNC`, `REFRESH` і `DELETE` усім користувачам незалежно від їхніх прав - UI не приховує дії за RBAC. Авторизація перевіряється сервером у момент виконання дії: спроба видалити застосунок під `dev1` завершиться помилкою `PermissionDenied`, а застосунок залишиться недоторканим.

---

## 8. Крок 6. Перевірка працездатності

Розгортання тестового застосунку підтверджує, що GitOps-цикл працює:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

argocd app sync guestbook
argocd app get guestbook
kubectl get pods -n default
```

Статус `Healthy` / `Synced` означає, що ArgoCD успішно прочитав маніфести з Git і привів стан кластера до бажаного.

![ArgoCD UI](../demo/argocd-ui.gif)

---

## 9. Демо-інструкція для команди

> Коротка інструкція, яку отримує розробник.

**Доступ до інтерфейсу ArgoCD:**

1. Відкрити в браузері **http://argocd.localhost:8080**
2. Увійти під виданим логіном (`dev1` / `dev2`) та паролем, отриманим від адміністратора
3. У списку застосунків обрати потрібний - доступні перегляд стану, дерева ресурсів, логів подів та ручна синхронізація

**Доступ через CLI:**

```bash
argocd login argocd.localhost:8080 --username dev1 --plaintext --grpc-web
argocd app list
argocd app get <app-name>
argocd app sync <app-name>
```

**Типові проблеми:**

| Симптом | Причина та рішення |
|---|---|
| Сторінка не завантажується | `kubectl get pods -n argocd` - усі поди мають бути `Running` |
| `argocd.localhost` не резолвиться | Додати запис у `hosts` (див. крок 3) |
| CLI повертає `404 Not Found` | Використати `--plaintext --grpc-web` замість `--insecure` |
| Попередження про сертифікат при вході | Збережений контекст має пріоритет - `argocd logout` і повторний вхід |
| `Invalid username or password` | Пароль для облікового запису ще не встановлено адміністратором |
| Нескінченний редирект у браузері | Перевірити `server.insecure` у `argocd-cmd-params-cm` |
| Кнопка `DELETE` доступна розробнику | Очікувана поведінка - UI не приховує дії, перевірка прав відбувається на сервері |
| Кластер недоступний | `k3d cluster list` - кластер має бути запущений |

---

## 10. Прибирання

```bash
k3d cluster delete asciiartify
```

---

## 11. Висновки

1. Розгортання ArgoCD на k3d технічно можливе та займає близько 5 хвилин від нуля до працездатного UI. PoC підтверджує: технічних перешкод для реалізації GitOps на обраному варіанті Kubernetes немає.
2. Вбудований у k3s Traefik дозволив налаштувати доступ до інтерфейсу без встановлення додаткового ingress-контролера - це підтверджує вибір k3d, зроблений на етапі Concept.
3. Основна нетривіальність - узгодження режимів TLS. Сервер переводиться в insecure-режим (`server.insecure: "true"`), інакше Traefik утворює цикл редиректів; відповідно CLI має підключатися з `--plaintext --grpc-web`, а не зі звичним `--insecure`.
4. Рольова модель ArgoCD дозволяє видати команді доступ до інтерфейсу без передавання облікового запису адміністратора. Обмеження перевірено командою `argocd account can-i`: розробник може синхронізувати застосунки, але не може їх створювати чи видаляти.
5. Система готова до реалізації MVP. Наступні кроки: підключення репозиторію AsciiArtify, опис застосунку як ArgoCD Application з `--sync-policy automated` та налаштування webhook для автоматичної синхронізації за подіями з Git.