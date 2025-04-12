# Инструкция-отчёт по запуску проекта

## Описание задания
Задание заключалось в улучшении безопасности приложения, замене Code Grant на PKCE и создании API для работы с отчётами. Требования:
1. Реализовать PKCE во фронтенде и Keycloak.
2. Создать бэкенд-часть приложения (выбран Node.js) с API `/reports`, который генерирует произвольные данные.
3. Бэкенд должен:
   - Отдавать данные только пользователям с ролью `prothetic_user`.
   - Проверять валидность подписи токена и возвращать `401 Unauthorized` при невалидной подписи.
4. Подготовить структуру проекта:
   - Папка `api` с кодом бэкенда и `Dockerfile`.
   - Обновить `docker-compose.yaml` для настройки и запуска API.
   - Обновить фронтенд для работы с PKCE flow.

---

## Структура проекта
После выполнения задания структура проекта следующая:

```
architecture-sprint-8/
├── api/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx
│   │   ├── index.tsx
│   │   └── components/
│   │       └── ReportPage.tsx
│   └── public/
├── keycloak/
│   └── realm-export.json
├── postgres-keycloak-data/
└── docker-compose.yaml
```

---

## Выполненные изменения

### 1. Реализация PKCE

#### Keycloak
Настроено PKCE для клиента `reports-frontend` в Keycloak Admin Console (`http://localhost:8080`, логин: `admin`, пароль: `admin`):
1. Открыта страница Keycloak Admin Console по адресу `http://localhost:8080`.
2. Введены учётные данные: логин `admin`, пароль `admin`.
3. В левой панели навигации выбран раздел **Clients**.
4. В списке клиентов найден и открыт клиент `reports-frontend`.
5. В разделе **Advanced settings** в поле **Proof Key for Code Exchange Code Challenge Method** установлено значение `S256` (метод SHA-256, рекомендованный для PKCE).
6. Проверено, что клиент `reports-frontend` настроен как публичный:
   - В разделе **Capability config** подтверждено, что **Client authentication** установлено в `OFF`.
   - **Standard flow** включён (это Authorization Code Flow, используемый с PKCE).

#### Фронтенд
Обновлён файл `frontend/src/App.tsx`, добавлены `initOptions` для включения PKCE:
   ```javascript
   const App: React.FC = () => {
     return (
       <ReactKeycloakProvider
         authClient={keycloak}
         initOptions={{
           onLoad: 'login-required',
           pkceMethod: 'S256', // Включён PKCE с методом S256
           timeout: 10000, // Тайм-аут 10 секунд
         }}
       >
         <div className="App">
           <ReportPage />
         </div>
       </ReactKeycloakProvider>
     );
   };
   ```


### 2. Разработка API

#### Создание кода API
Выбран Node.js для разработки API.

1. **Инициализация проекта**:
   - Внутри директории `api` создан файл `package.json` для управления зависимостями:
     ```json
     {
       "name": "api",
       "version": "1.0.0",
       "main": "server.js",
       "dependencies": {
         "express": "^4.18.2",
         "@keycloak/keycloak-admin-client": "^22.0.1",
         "jsonwebtoken": "^9.0.2",
         "jwks-rsa": "^3.1.0",
         "cors": "^2.8.5"
       }
     }
     ```
   - Установлены зависимости с помощью команды `npm install`, указанные в `Dockerfile` для автоматической установки при сборке.

2. Внутри `api` создан `Dockerfile` для сборки API:
   ```dockerfile
   FROM node:16
   WORKDIR /app
   COPY package.json .
   RUN npm install express @keycloak/keycloak-admin-client jsonwebtoken jwks-rsa cors
   COPY . .
   CMD ["node", "server.js"]
   ```

3. **Создание основного кода**:
   - Создан файл `api/server.js` с базовой структурой Express-сервера:
     ```javascript
     const express = require('express');
     const cors = require('cors');

     const app = express();

     app.use(cors({
       origin: 'http://localhost:3000',
       methods: ['GET', 'POST', 'OPTIONS'],
       allowedHeaders: ['Content-Type', 'Authorization'],
     }));

     app.use(express.json());
     ```
   - Добавлено логирование всех входящих запросов для отладки:
     ```javascript
     app.use((req, res, next) => {
       console.log(`Received ${req.method} request to ${req.url}`);
       next();
     });
     ```

4. **Настройка Keycloak**:
   - Настроено подключение к Keycloak для проверки токенов:
     ```javascript
     const keycloakRealm = 'reports-realm';
     const keycloakUrl = 'http://keycloak:8080';
     const issuerUrl = 'http://localhost:8080';
     const jwksUri = `${keycloakUrl}/realms/${keycloakRealm}/protocol/openid-connect/certs`;

     console.log(`JWKS URI: ${jwksUri}`);

     const client = jwksClient({
       jwksUri: jwksUri,
     });

     function getKey(header, callback) {
       client.getSigningKey(header.kid, (err, key) => {
         if (err) {
           console.error('Error fetching signing key:', err);
           callback(err, null);
         } else {
           const signingKey = key?.getPublicKey();
           callback(null, signingKey);
         }
       });
     }
     ```
   - `keycloakUrl` оставлено как `http://keycloak:8080`, так как это используется внутри Docker-сети, а `issuerUrl` установлено в `http://localhost:8080`.

5. **Middleware для проверки токенов**:
   - Написан middleware `verifyToken` для проверки токена и роли:
     ```javascript
     const verifyToken = (req, res, next) => {
       const authHeader = req.headers['authorization'];
       if (!authHeader) {
         console.log('No Authorization header provided for', req.url);
         return res.status(401).json({ error: 'No token provided' });
       }

       const token = authHeader.split(' ')[1];
       if (!token) {
         console.log('Invalid token format for', req.url);
         return res.status(401).json({ error: 'Invalid token format' });
       }

       console.log('Token for', req.url, ':', token);

       jwt.verify(token, getKey, {
         issuer: `${issuerUrl}/realms/${keycloakRealm}`,
         algorithms: ['RS256'],
       }, (err, decoded) => {
         if (err) {
           console.error('Token verification failed for', req.url, ':', err.message);
           return res.status(401).json({ error: 'Token verification failed', details: err.message });
         }

         if (decoded.azp !== 'reports-frontend') {
           console.log('Invalid azp for', req.url, ':', decoded.azp);
           return res.status(401).json({ error: 'Invalid client', details: 'azp does not match reports-frontend' });
         }

         if (!decoded.realm_access?.roles.includes('prothetic_user')) {
           console.log('User does not have required role for', req.url, ':', decoded.realm_access?.roles);
           return res.status(403).json({ error: 'Insufficient permissions', details: 'prothetic_user role required' });
         }

         console.log('Decoded token for', req.url, ':', decoded);
         req.user = decoded;
         next();
       });
     };
     ```
   - Middleware проверяет токен, клиент (`azp`), роль `prothetic_user` и возвращает соответствующие ошибки (`401` или `403`).

6. **Эндпоинт `/reports`**:
   - Добавлен эндпоинт `/reports` для генерации случайных данных:
     ```javascript
     app.get('/reports', verifyToken, (req, res) => {
       const report = {
         id: Math.floor(Math.random() * 1000),
         title: `Usage Report ${Math.floor(Math.random() * 100)}`,
         content: `This is a sample report generated on ${new Date().toISOString()}.`,
         user: req.user.preferred_username,
       };
       res.json(report);
     });
     ```
   - Эндпоинт генерирует случайный `id`, `title` и `content`, а также добавляет имя пользователя из токена.

#### Полный код `server.js`
Полный код файла `api/server.js`, который реализует API:

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: 'http://localhost:3000',
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
}));

app.use(express.json());

// Логирование всех входящих запросов
app.use((req, res, next) => {
  console.log(`Received ${req.method} request to ${req.url}`);
  next();
});

// Настройка Keycloak
const keycloakRealm = 'reports-realm';
const keycloakUrl = 'http://keycloak:8080';
const issuerUrl = 'http://localhost:8080';
const jwksUri = `${keycloakUrl}/realms/${keycloakRealm}/protocol/openid-connect/certs`;

console.log(`JWKS URI: ${jwksUri}`);

const client = jwksClient({
  jwksUri: jwksUri,
});

function getKey(header, callback) {
  client.getSigningKey(header.kid, (err, key) => {
    if (err) {
      console.error('Error fetching signing key:', err);
      callback(err, null);
    } else {
      const signingKey = key?.getPublicKey();
      callback(null, signingKey);
    }
  });
}

const verifyToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  if (!authHeader) {
    console.log('No Authorization header provided for', req.url);
    return res.status(401).json({ error: 'No token provided' });
  }

  const token = authHeader.split(' ')[1];
  if (!token) {
    console.log('Invalid token format for', req.url);
    return res.status(401).json({ error: 'Invalid token format' });
  }

  console.log('Token for', req.url, ':', token);

  jwt.verify(token, getKey, {
    issuer: `${issuerUrl}/realms/${keycloakRealm}`,
    algorithms: ['RS256'],
  }, (err, decoded) => {
    if (err) {
      console.error('Token verification failed for', req.url, ':', err.message);
      return res.status(401).json({ error: 'Token verification failed', details: err.message });
    }

    if (decoded.azp !== 'reports-frontend') {
      console.log('Invalid azp for', req.url, ':', decoded.azp);
      return res.status(401).json({ error: 'Invalid client', details: 'azp does not match reports-frontend' });
    }

    if (!decoded.realm_access?.roles.includes('prothetic_user')) {
      console.log('User does not have required role for', req.url, ':', decoded.realm_access?.roles);
      return res.status(403).json({ error: 'Insufficient permissions', details: 'prothetic_user role required' });
    }

    console.log('Decoded token for', req.url, ':', decoded);
    req.user = decoded;
    next();
  });
};

// Эндпоинт /reports с генерацией случайных данных
app.get('/reports', verifyToken, (req, res) => {
  const report = {
    id: Math.floor(Math.random() * 1000), // Случайный ID
    title: `Usage Report ${Math.floor(Math.random() * 100)}`,
    content: `This is a sample report generated on ${new Date().toISOString()}.`,
    user: req.user.preferred_username,
  };
  res.json(report);
});

app.listen(8000, () => console.log('Backend running on port 8000'));
```

#### Обновление `docker-compose.yaml`
1. Добавлен сервис `api` для реализации API:
   - Добавлен новый сервис `api` с указанием сборки из директории `api`:
     ```yaml
     api:
       build:
         context: ./api
         dockerfile: Dockerfile
       ports:
         - "8000:8000"
       depends_on:
         - keycloak
     ```
   - В сервисе `frontend` добавлена зависимость от `api` через `depends_on`, а также установлена переменная окружения `REACT_APP_API_URL` для подключения к API:
     ```yaml
     frontend:
       build:
         context: ./frontend
         dockerfile: Dockerfile
       ports:
         - "3000:3000"
       environment:
         REACT_APP_API_URL: http://localhost:8000
         REACT_APP_KEYCLOAK_URL: http://localhost:8080
         REACT_APP_KEYCLOAK_REALM: reports-realm
         REACT_APP_KEYCLOAK_CLIENT_ID: reports-frontend
       depends_on:
         - api
         - keycloak
     ```

---

## Инструкция по запуску проекта

### Предварительные требования
- Проверить: порты `3000`, `8000`, и `5433` свободны:
  ```bash
  netstat -aon | findstr :3000
  netstat -aon | findstr :8000
  netstat -aon | findstr :5433
  ```
  Если порты заняты, необходимо остановить процессы:
  ```bash
  taskkill /PID <PID> /F
  ```

### Необходимые шаги для запуска

1. Запустить проект:
   ```bash
   docker compose up -d --build
   ```

2. Проверить статус сервисов:
   ```bash
   docker compose ps
   ```
   Необходимо убедиться, что все сервисы запущены:
   ```
   NAME                    COMMAND                  SERVICE             STATUS              PORTS
   sprint-8-api-1          "docker-entrypoint.s…"   api                 Up                  0.0.0.0:8000->8000/tcp
   sprint-8-frontend-1     "docker-entrypoint.s…"   frontend            Up                  0.0.0.0:3000->3000/tcp
   sprint-8-keycloak-1     "/opt/keycloak/bin/k…"   keycloak            Up                  0.0.0.0:8080->8080/tcp
   sprint-8-keycloak_db-1  "docker-entrypoint.s…"   keycloak_db         Up                  0.0.0.0:5433->5432/tcp
   ```

3. Проверить работу приложения:
   - Необходимо открыть `http://localhost:3000` в браузере.
   - Пройти перенаправление на страницу входа Keycloak.
   - Войти как пользователь `prothetic1` (пароль: `password123`).
   - Нажать "Download Report".
   - Убедиться, что отображается отчёт с случайными данными:
     ```json
     {
       "id": 542,
       "title": "Usage Report 73",
       "content": "This is a sample report generated on 2025-04-06T12:34:56.789Z.",
       "user": "prothetic1"
     }
     ```

### Остановка проекта
Чтобы остановить проект, необходимо выполнить:
```bash
docker compose down
```

---

## Проверка требований
- **PKCE**: Реализован во фронтенде и Keycloak.
  - Подтверждено, что PKCE работает корректно: запросы к Keycloak содержат параметры `code_challenge` и `code_challenge_method=S256`. Для проверки открыт `http://localhost:3000`, в инструментах разработчика (F12 → Network) проверено наличие параметров `code_challenge` и `code_challenge_method=S256` в запросе к `/realms/reports-realm/protocol/openid-connect/auth`.
- **API `/reports`**:
  - Создан в `api/server.js`, генерирует случайные данные.
  - Доступ ограничен для пользователей с ролью `prothetic_user` (иначе возвращает `403 Forbidden`).
  - Проверяет подпись токена и возвращает `401 Unauthorized` при невалидной подписи.
  - Отправлен запрос к `http://localhost:8000/reports` с валидным токеном пользователя `prothetic1` (с ролью `prothetic_user`):
    ```json
    {
      "id": 542,
      "title": "Usage Report 73",
      "content": "This is a sample report generated on 2025-04-06T12:34:56.789Z.",
      "user": "prothetic1"
    }
    ```
  - Проверено с пользователем без роли `prothetic_user` (например, `user1`):
    - Получен ответ `403 Forbidden`:
      ```json
      { "error": "Insufficient permissions", "details": "prothetic_user role required" }
      ```
  - Проверено с невалидным токеном:
    - Получен ответ `401 Unauthorized`:
      ```json
      { "error": "Token verification failed", "details": "invalid token" }
      ```
- **Структура проекта**:
  - Папка `api` содержит `server.js`, `Dockerfile` и `package.json`.
  - `docker-compose.yaml` обновлён для настройки и запуска API, добавлен сервис `api` и зависимость от него в `frontend`.
  - Фронтенд обновлён для работы с PKCE flow.

---

