# ASP.NET Core + React: Создание монолитного клиент-серверного приложения с аутентификацией и авторизацией с помощью пакетов TourmalineCore

## Вступление

Реализация процесса аутентификации в приложениях является одной из самых распространённых задач в Web-разработке. Большинство существующих решений опираются только на свою сторону клиент-серверного взаимодействия, предоставляя второй стороне максимально обобщенные интерфейсы для работы.

При частом создании подобных приложений, сооружение фасадов между клиентом и сервером может стать рутинным занятием. В такой ситуации могут помочь библиотеки, версии которых существуют по обе стороны баррикад и хорошо заточены под работу друг с другом. 

Данная статья описывает создание монолитного приложения на платформах ASP.NET Core и React на примере пакетов TourmalineCore: [TourmalineCore.AspNetCore.JwtAuthentication.Identity](https://github.com/TourmalineCore/TourmalineCore.AspNetCore.JwtAuthentication/tree/master/JwtAuthentication.Identity) и [@tourmalinecore/react-tc-auth](https://www.npmjs.com/package/@tourmalinecore/react-tc-auth).

# Back

## Основные преимущества пакета TourmalineCore.AspNetCore.JwtAuthentication.Identity:
- Гибкий набор middleware для реализации Аутентификации, Авторизации, Регистрации.
- Аутентификация на основе JWT в качестве Access-токенов и RSA в качестве  механизма подписи.
- Использование Refresh-токена для увеличения безопасности за счет снижения времени жизни Access-токенов.
- Возможность использования fingerprint'ов. [Подробнее](#fingerprint)
- Использование Microsoft.AspNetCore.Identity для создания моделей и EntityFrameworkCore для работы с базой данных.

## Создание проекта

> Для работы потребуется [.NET Core](https://dotnet.microsoft.com/download) версии 3.0 и выше, а также [Visual Studio](https://visualstudio.microsoft.com/ru/).

1. Создаём новый проект ASP.NET Core Web Application версии 3.0 или выше. В качестве шаблона нужно выбрать ASP.NET Core Empty.

2. Добавляем в проект необходимые Nuget пакеты:
    - **TourmalineCore.AspNetCore.JwtAuthentication.Identity**
    - **Microsoft.EntityFrameworkCore.InMemory**. Необходимо для создания БД. Для демонстрации нам достаточно будет InMemory-базы.

3. Создаём модель пользователя, унаследовав её от класса IdentityUser из пакета Microsoft.AspNetCore.Identity. 

> Вы также можете не создавать свой класс пользователя и использовать непосредственно класс IdentityUser, если вас устраивает его набор полей.

```csharp
using Microsoft.AspNetCore.Identity;

namespace AuthExample.Models
{
    public class CustomUser : IdentityUser
    {
        public string ZipCode { get; set; }
    }
}
```

4. Создаём DbContext, унаследовав его от TourmalineDbContext и передав класс своего пользователя в качестве generic-параметра.

```csharp
using AuthExample.Models;
using Microsoft.EntityFrameworkCore;
using TourmalineCore.AspNetCore.JwtAuthentication.Identity;

namespace AuthExample.Data
{
    public class AppDbContext : TourmalineDbContext<CustomUser>
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options)
        {
        }
    }
}
```

5. Обновляем Startup.cs следующим образом:
```csharp
using AuthExample.Data;
using AuthExample.Models;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using TourmalineCore.AspNetCore.JwtAuthentication.Core;
using TourmalineCore.AspNetCore.JwtAuthentication.Core.Options;
using TourmalineCore.AspNetCore.JwtAuthentication.Identity;
using TourmalineCore.AspNetCore.JwtAuthentication.Identity.Options;

namespace AuthExample
{
    public class Startup
    {
        private readonly IConfiguration _configuration;

        public Startup(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        public void ConfigureServices(IServiceCollection services)
        {
            // Для демонстрации 
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("Database")
            );

            // Разрешаем зависимости middleware
            var authenticationOptions = (_configuration.GetSection(nameof(AuthenticationOptions)).Get<RefreshAuthenticationOptions>());
            services
                .AddJwtAuthenticationWithIdentity<AppDbContext, CustomUser>()
                .AddLoginWithRefresh(authenticationOptions)
                .AddLogout();

            services.AddControllers();
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            // Настраиваем CORS. Это необходимо, так как два наших приложения технически будут располагаться в разных доменах
            app.UseCors(
                builder => builder
                    .AllowAnyHeader()
                    .SetIsOriginAllowed(host => true)
                    .AllowCredentials()
                    .AllowAnyMethod()
            );

            app.UseRouting();

            // Добавляем пользователя, которого будем использовать для тестов
            app.UseDefaultDbUser<AppDbContext, CustomUser>("Admin", "Admin");

            // Подключаем middleware
            app
                .UseJwtAuthentication()
                .UseDefaultLoginMiddleware()
                .UseRefreshTokenMiddleware()
                .UseRefreshTokenLogoutMiddleware();

            app.UseEndpoints(endpoints => { endpoints.MapControllers(); });
        }
    }
}

```

6. Обновляем appsettings.json, добавив в него ключи RSA. Вы можете использовать пару ключей из примера или сгенерировать их самостоятельно. Для генерации данного примера использовался [вот этот сайт](https://travistidwell.com/jsencrypt/demo/).

```json
{
  "AuthenticationOptions": {
    "PublicSigningKey": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsDwLnM5sbVi326YDsLvMkQLXDKVAaHrJZ/MwkoxF4Hmq4+pu4KojgQyVDtjseXG8UW5wbxW58eXG8V0XgJzsD8zQX2Z1bBawpIeD9sXf/5CFZGif85YFIqS3brqR3ScdGxYHXcwrUMGUCThxe918Q0aNXzdSxGGP2v7ZbtpFhLRyrTXHl4u6k3eyYG7zCkwextnMb9CJuCR7x1ua1V1S0xljAqg5PicFjt0vVSKzPM/Djw7XK84sJXxaet7t4cNtXVJIAyXUMsSli6gg9Cw9CEUSE40iWUR/6wrdUYAchk3vWiBhMmnufwzmFRLKHOH9Fz8buJVSrRfyt7a6S2iN+wIDAQABMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsDwLnM5sbVi326YDsLvMkQLXDKVAaHrJZ/MwkoxF4Hmq4+pu4KojgQyVDtjseXG8UW5wbxW58eXG8V0XgJzsD8zQX2Z1bBawpIeD9sXf/5CFZGif85YFIqS3brqR3ScdGxYHXcwrUMGUCThxe918Q0aNXzdSxGGP2v7ZbtpFhLRyrTXHl4u6k3eyYG7zCkwextnMb9CJuCR7x1ua1V1S0xljAqg5PicFjt0vVSKzPM/Djw7XK84sJXxaet7t4cNtXVJIAyXUMsSli6gg9Cw9CEUSE40iWUR/6wrdUYAchk3vWiBhMmnufwzmFRLKHOH9Fz8buJVSrRfyt7a6S2iN+wIDAQAB",
    "PrivateSigningKey": "MIIEowIBAAKCAQEAsDwLnM5sbVi326YDsLvMkQLXDKVAaHrJZ/MwkoxF4Hmq4+pu4KojgQyVDtjseXG8UW5wbxW58eXG8V0XgJzsD8zQX2Z1bBawpIeD9sXf/5CFZGif85YFIqS3brqR3ScdGxYHXcwrUMGUCThxe918Q0aNXzdSxGGP2v7ZbtpFhLRyrTXHl4u6k3eyYG7zCkwextnMb9CJuCR7x1ua1V1S0xljAqg5PicFjt0vVSKzPM/Djw7XK84sJXxaet7t4cNtXVJIAyXUMsSli6gg9Cw9CEUSE40iWUR/6wrdUYAchk3vWiBhMmnufwzmFRLKHOH9Fz8buJVSrRfyt7a6S2iN+wIDAQABAoIBAQCvue/KV3p+Pex2tD8RxvDf13kfPtfOVkDlyfQw7HXwsuDXijctBfmJAEbRGzQQlHw2pmyuF3fl4DxTB4Qb1lz8FDniJoQHV0ijhgzrz7rfVffsevajKH/OX3gYjShM4GeBTqHhwWefiqZV21YtMFhrrLniq4N4FeAfeebNRg/zlWEigraxqAWb4cplnxBE3qOBECKXdF/B8uhp743BU/2HLSO5BUdhtPlN3FKoYdyqtrKyNO2z7rC+Gk8tNd+KbMHDUMiOQXzbXkpsXYKAug9iTW+gxZG/bNyzGNrJBFrUYb1fP4iZphbxBJgobNYJBKA565cAX/wI5lFakTBB0YAhAoGBAOk0TyV0dA8WJ6NrWmRUBKsKvkSREhBveW+P3LtA8a1IgQf4K6ohIfcq9w/+nRvTLPIxo67FcqEyzVUu9TOafzIi59w4RBWG/HKOZ5lvIVicbuPyclPVWyC+9bMMgWEJy9wGwE+fGh3AvAA4PXNBcjOqfT0sSF9PBUo5qN11Q/qHAoGBAMF2IL+cXgPiUta4XoMh14ksJiwHtZeMkj+kauU3rctDITSkIGMFp4q0W5UUSG1yPcW/++rMQfuAjCZotdNpbQT+g+KfG44DMT5W7nRgv60S0/6X/OoLIhCue19yLMVzFpai0YEH+s24/XNnwl53K34G1zVMCsZcIuIng8SZVintAoGAJP/1pr2pRFOBin4X418pNnIH6h0SPqVRIRA0N0mAjru4LSmE1ANZvjuE43bEOovwz6Rskegl3cmPpnpC0SMsFypOmzQaKUg3eX16lm95XPPE7EmlNgPd534kwXm0dU72lzxC+t8FZ78SlP5XUZgKpIPiRvhlqymAb1xinHBkjrUCgYAB144YRPTgNJd1U+wSc5AJzlHOuYQRHVWHJZme9RjChrEaPzXPu44M1ArLMJY/9IaCC4HqimdWbbLn6rdQfAB9u66lyb4JbB5b6Zf7o7Avha5fDjNqRxDb981U61Fhz+a3KHW2NM0+iDRhlOtU2u2fFZGXAFJZ8Saj4JxwksUvQQKBgEQ1TAW/INhWSkEW8vGeLnjV+rxOx8EJ9ftVCRaQMlDEDlX0n7BZeQrQ1pBxwL0FSTrUQdD02MsWshrhe0agKsw2Yaxn8gYs1v9HMloS4Q3L2zl8pi7R3yx72RIcdnS4rqGXeO5t8dm305Yz2RHhqtkBmpFBssSEYCY/tUDmsQVU"
  }
}
```

7. Последним файлом создаём контроллер, который будет использоваться для проверки работы Access-токена.

```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace AuthExample.Controllers
{
    [ApiController]
    [Authorize]
    [Route("[controller]")]
    public class ExampleController : ControllerBase
    {
        [HttpGet]
        public IEnumerable<object> Get()
        {
            return new []{
                "Freezing",
                "Bracing",
                "Chilly",
                "Cool",
                "Mild",
                "Warm",
                "Balmy",
                "Hot",
                "Sweltering",
                "Scorching",
            };
        }
    }
}
```
> **Важно**: предполагается, что Back будет запущен на 5000-ом порту. Это порт, присваемый новым проектам в VS по умолчанию. Если вы хотите изменить это значение, то не забудьте внести правки в код в фронтовом проекте. 

## Эндпоинты

Если всё сделано правильно, то при запуске приложения нам станут доступны следующие эндпоинты:

POST http://localhost:5000/auth/login 

Content-Type: application/json

Тело запроса:
```json
{
    "login": "Admin", 
    "password": "Admin", 
    "clientFingerPrint": "fingerprint"
}
```

POST http://localhost:5000/auth/refresh

Content-Type: application/json

Тело запроса:
```json
{
    "refreshTokenValue": "<рефреш-токен>",
    "clientFingerPrint": "fingerprint"
}
```

POST http://localhost:5000/auth/logout

Content-Type: application/json

Тело запроса:
```json
{
    "refreshTokenValue": "<рефреш-токен>",
    "clientFingerPrint": "fingerprint"
}
```

Успешные запросы на логин и рефреш вернут JSON в следующем виде:
```json
{
    "accessToken": {
        "value": "{{ACCESS_TOKEN_VALUE}}",
        "expiresInUtc": "2021-01-01T00:00:00.0000000Z"
    },
    "refreshToken": {
        "value": "{{REFRESH_TOKEN_VALUE}}",
        "expiresInUtc": "2021-01-01T00:00:00.0000000Z"
    }
}
```

## FingerPrint

В некоторых запросах присутствует параметр `clientFingerPrint`. Если вкратце, то **fingerprint** - это инструмент отслеживания браузера вне зависимости от желания пользователя быть идентифицированным. Обчыно это хеш, сгенерированный js'ом на базе неких уникальных параметров/компонентов браузера. Преимущество fingerprint'a состоит в том, что он нигде не хранится постоянно и генерируется только в момент логина и рефреша. ([источник](https://gist.github.com/zmts/802dc9c3510d79fd40f9dc38a12bccfc))

# Front

## Описание пакета react-tc-auth:
- Создание запросов к API аутентификации, включая Логин, Рефреш и Логаут.
- Быстрый доступ к токенам

## Создание проекта

> Для работы потребуется [node.js](https://nodejs.org/en/). 

1. Создаём заготовку React-приложения
```
npx create-react-app auth-example-front
```

2. Переходим в созданный каталог 
```
cd auth-example-front
```

3. Устанавливаем пакет [@tourmalinecore/react-tc-auth](https://www.npmjs.com/package/@tourmalinecore/react-tc-auth)
```
npm i react-router-dom --save
npm i @tourmalinecore/react-tc-auth --save
```

4. Из файлов приложения прежде всего нужно инициализировать **authService**, что мы и делаем, используя функцию из загруженного пакета.

> Пути файлов, приведенных далее, соответсвуют шаблону "auth-example-front/src/название файла"

**services/authService.js**
```JSX
import { createAuthService } from '@tourmalinecore/react-tc-auth';

export const authService = createAuthService({
  authApiRoot: 'http://localhost:5000/auth', // путь к серверу аутентификации
  authType: 'ls', // тип, определяющий, где будут хранится токены. в данном случае, Local Storage

  // аксесоры параметров для объектов, которые приложение получает с бэка
  tokenAccessor: 'accessToken',
  refreshTokenAccessor: 'refreshToken',
  tokenValueAccessor: 'value',
  tokenExpireAccessor: 'expiresInUtc',
});
```

5. Далее authService будет использован для создания нашего api-клиента, который будет применяться для выполнения запросов. С помощью интерсепторов axios мы  автоматизируем два сценария: добавлление Auth-токена к каждому запросу и обновление токена при его просрочке.

**services/api.js**
```JSX
import axios from 'axios';

import { authService } from './authService';

export const api = axios.create({
  baseURL: 'http://localhost:5000',
});

api.interceptors.request.use((config) => {
  const token = authService.getAuthToken();

  // добавляем токен в хедеры
  config.headers.Authorization = token ? `Bearer ${token}` : '';

  return config;
}, null);

api.interceptors.response.use(null, async (error) => {
  // если запрос провалился из-за аутентификации или авторизации, пробуем обновить access-токен с помощью refresh-токена 
  if (
    (error.response && error.response.status === 401)
    || (error.response && error.response.status === 403)
  ) {
    // если токена нет, считаем, что пользователь не вошел в систему, и никаких действий предпринимать не нужно
    if (!authService.getAuthToken()){
      return Promise.reject(error);
    }

    // запрашиваем обновление токена
    await authService.refreshToken();

    // переопределяем токен в хедерах старого запроса 
    const token = authService.getAuthToken();
    error.config.headers.Authorization = token ? `Bearer ${token}` : '';

    // повторяем запрос
    return api.request(error.config);
  }

  return Promise.reject(error);
});

```

6. Теперь нам нужно обернуть все компоненты приложения в контекст сервиса аутентификации, чтобы они могли использовать его данные.

**index.js**
```JSX
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import { authService } from './services/authService';

ReactDOM.render(
  <React.StrictMode>
     <authService.AuthProvider>
      <App />
    </authService.AuthProvider>
  </React.StrictMode>,
  document.getElementById('root')
);
```

7. И наконец, мы можем создать страницу, используя полученные средства для работы с API сервера.

**App.jsx**
```JSX
import { useContext, useState } from 'react';
import { authService } from './services/authService';
import { api } from './services/api';

export default function Authentication(){
  // использование контекста сервиса аутентификации
  const [isAuthenticated] = useContext(authService.AuthContext);

  const [login, setLogin] = useState('');
  const [password, setPassword] = useState('');
  const [data, setData] = useState('');
  const [loading, setLoading] = useState(false);

  return (
    <div>
      {
        isAuthenticated
          ? (
            <div>
              Вы вошли в систему
              <br />
              <br />
              <button
                style={{
                  marginRight: 16,
                }}
                type="button"
                onClick={logoutUser}
              >
                Выйти
              </button>

              <button
                type="button"
                onClick={setExpired}
              >
                Просрочить токен
              </button>
            </div>
          )
          : (
            <div>
              Логин
              <br />
              <input value={login} onChange={(e) => setLogin(e.target.value)} />
              <br />
              Пароль
              <br />
              <input value={password} onChange={(e) => setPassword(e.target.value)} />
              <br />
              <br />
              <button type="button" onClick={sendLoginData}>Войти</button>
            </div>
          )
      }
      <br />
      <br />
      <button
        type="button"
        onClick={getData}
      >
        Получить данные
      </button>
      <br />
      <br />
      {loading ? 'Загрузка...' : data}
    </div>
  );

  function sendLoginData() {
    authService.loginCall({
      login,
      password,
    })
      .then((response) => authService.setLoggedIn(response.data));
  }

  function logoutUser() {
    authService.logoutCall();
    authService.setLoggedOut();
  }

  function setExpired() {
    localStorage.setItem('accessToken', JSON.stringify({
      value: '12345',
      expiresInUtc: '2010-04-19T06:43:27.2953284Z',
    }));
  }

  async function getData() {
    setLoading(true);
    try {
      const response = await api.get('/example');
      setData(response.data.map(x => <p key={x}>{x}</p>));
    }
    catch {
      setData('Ошибка');
    }
    finally{
      setLoading(false);
    }
  }
};

```

# Использование

1. Запускаем оба проекта.
2. В окне фронтового проекта видим форму логина.
3. Открываем консоль браузера, чтобы видеть статусы запросов.
4. Пытаемся получить данные, будучи неавторизованными. Закономерно получаем 401.
5. Логинимся с неправильными кредами - 401.
6. Теперь пробуем правильные (Admin/Admin) и входим в систему.
7. Снова пробуем получить данные, в этот раз успешно.
8. Для проверки рефреша нажимаем "Просрочить токен".
9. Еще раз жмём "Получить данные". В запросах видим, что сначала вернулся 401-ый код ошибки, но благодаря интерсептору был сразу же отправлен запрос на обновление токена, после чего повторный запрос таки вернул нам наши данные. 
10. Нажимаем "Выход" и удостоверяемся, что токены были сброшены.
