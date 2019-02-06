# 3. API-интерфейсы GraphQL

HTTP-сервер — наиболее распространённый способ предоставить схему GraphQL. Создание API-интерфейсов GraphQL — это гораздо больше, чем просто проектирование схем. В этой главе вы узнаете, как создавать надежные, многоуровневые API-интерфейсы GraphQL.

![Сервер](images/server.png)

Вы узнаете, как предоставлять GraphQL-схемы с помощью Apollo Server. Как подключить ваши резолверы к базе данных. Вы добавите аутентификацию по электронной почте в собственный API. Наконец, вы научитесь, как организовать исходный код по функциональным возможностям.

![Слои сервера](images/server-layers.png)

Все этапы этой главы имеют соответствующий проект, который можно склонировать, чтобы попробовать на практике.

Давайте начнем с изучения того, как создать API, используя Apollo Server.

## 3.1 Сервер

Apollo Server — это сервер с открытым исходном кодом, совместимый со спецификацией GraphQL. Это готовый к работе в продакшене, простой в настройке способ представления GraphQL-схем, так чтобы HTTP-клиенты могли их использовать.

![HTTP](images/http.png)

Вы уже создали схему с использованием `graphql-tools` в предыдущей главе, поэтому публично представить её с помощью Apollo Server не составляет никакого труда.

```js
const { ApolloServer } = require("apollo-server");

const schema = require("./schema");

const server = new ApolloServer({ schema });

server.listen().then(({ url }) => {
  console.log(`🚀  Server ready at ${url}`);
});
```

Действительно просто! Достаточно вызова функции `server.listen()` и у вас есть работающий API для GraphQL. Склонируйте следующий пример проекта, чтобы у вас была собственная копия проекта.

[Клонировать пример сервера](https://glitch.com/edit/#!/remix/pinapp-server)

Нажмите кнопку «Show» в левом верхнем углу экрана для открытия включенного в комплект GraphQL-клиента — [GraphQL Playground](https://github.com/prismagraphql/graphql-playground). Это больше, чем просто клиент GraphQL, скорее что-то вроде IDE. У этого инструмента есть автозаполнение запроса, документация GraphQL-схемы, а также хранение всех выполненных запросов, что позволяет использовать в будущем.

![GraphQL Playground](images/graphql-playground.png)

Теперь, когда вы развернули собственный API-сервер GraphQL, пришло время добавить сохранение данных, используя базу данных.

## 3.2 База данных

API-интерфейсы GraphQL могут быть работать с любым источником данных. Они могут использовать базы данных SQL, NoSQL, базы данных в оперативной памяти или даже использовать конечные точки HTTP.

![База данных](images/database.png)

В этой главе вы подключите PinApp к базе данных SQLite, используя коннектор базы данных [Knex](http://knexjs.org). Knex — это построитель SQL-запросов, который может взаимодействовать со многими базами данных SQL, такими как SQLite3, MySQL, Postgres и др.

Склонируйте текущую итерацию приложения PinApp, чтобы вы могли следовать по тексту этого раздела, работая с собственной копией проекта.

[Клонировать пример базы данных](https://glitch.com/edit/#!/remix/pinapp-database)

> Не забывайте следовать инструкциям по началу работы в файле README проекта

Поскольку все взаимодействия с базами данных происходят в файле `resolvers.js`, давайте начнем с этого файла.

Получить набор записей с помощью `select()`. Например, получить список пинов можно следующим образом:

```js
pins: () => database("pins").select(),
```

Вы можете объединить в цепочку вызовов функции Knex. Например, можно отфильтровать результаты из `select()`, добавив его вызов с функцией `where()`.

```js
search: (_, { text }) => {
  return Promise.all([
    database("users")
      .select()
      .where("email", "like", `%${text}%`),
    database("pins")
      .select()
      .where("title", "like", `%${text}%`)
  ]);
};
```

Еще одна полезная функция, которую предоставляет Knex, — это `insert()`. Она позволит создавать объекты в базе данных. Вот так выглядит мутация `addPin` при использовании `insert`.

```js
addPin: async (_, { pin }, { token }) => {
  const [user] = await authorize(token);
  const { user: updatedUser, pin: createdPin } = await addPin(user, pin);
  await database("pins").insert(createdPin);
  return createdPin;
},
```

Файл `database.js` создает экземпляр Knex и экспортирует в качестве модуля. Это простой файл, который создает экземпляр Knex с конфигурационными значениями из файла с именем `knexfile.js`.

```js
const database = require("knex")(require("./knexfile"));

module.exports = database;
```

У Knex имеется интерфейс командной строки, предоставляющий несколько утилит для создания миграций, заполнения данными баз данных и многое другое. Утилита CLI считывает конфигурацию из `knexfile.js`. Файл конфигурации PinApp выглядит так:

```js
module.exports = {
  client: "sqlite3",
  connection: {
    filename: ".data/database.sqlite"
  }
};
```

> [Glitch](https://glitch.com) позволяет сохранять данные в директории `.data`. Именно по этой причине файл `database.sqlite` находится в этой директории.

Последний шаг, необходимый для использования SQL-файла, — создание схемы базы данных. Knex может генерировать файлы миграции, используя собственный инструмент командной строки.

Выполнение `npx knex migrate:make create_users_table` создаст файл с именем `[date]_create_users_table.js` в директории `.migrations`. Этот файл экспортирует два метода — `up` и `down`. Эти методы являются своего рода заполнителями, которые необходимо заполнить согласно ваших конкретным потребностям. В данном случае таблица пользователей будет иметь два поля: `id` и `email`. У обоих полей указан тип `string`, а поле `id` задано как первичный ключ.

```js
exports.up = function(knex) {
  return knex.schema.createTable("users", function(table) {
    table.string("id").primary();
    table.string("email");
  });
};

exports.down = function(knex) {
  return knex.schema.dropTable("users");
};
```

В склонированном проекте есть еще одна миграция — `[date]_create_pins_migration`. Эта миграция определяет пять строковых (`string`) полей: `id`, `title`, `link`, `image` и `pin_id`.

```js
exports.up = function(knex) {
  return knex.schema.createTable("pins", function(table) {
    table.string("id").primary();
    table.string("title");
    table.string("link");
    table.string("image");
    table
      .string("user_id")
      .references("id")
      .inTable("users")
      .onDelete("CASCADE")
      .onUpdate("CASCADE");
  });
};

exports.down = function(knex) {
  return knex.schema.dropTable("pins");
};
```

Выполните `npm run setup-db` для применения всех миграций базы данных. Этот скрипт определен в ключе `scripts` файла `package.json`:

```json
"setup-db": "knex migrate:latest"`
```

Изучение SQL выходит за рамки книги: если вы хотите правильно изучить этот язык, лучше всего прочитать соответствующую книгу. Построитель запросов Knex отлично работает с базами данных SQL, а еще у него есть отличная [документация](http://knexjs.org). Обратитесь к ней, если вы хотите узнать больше про данный инструмент.

## 3.3 Аутентификация

Один из самых часто задаваемых вопросов при создании API-интерфейсов с использованием GraphQL: «Куда поместить аутентификацию и авторизацию?». Это должно быть в слое GraphQL? Или на уровне базы данных? А может быть в бизнес-логики? Несмотря на то, что ответ зависит от контекста того, какой API вы создаете, распространенный способ решения этой проблемы — поместить аутентификацию и авторизацию на бизнес-уровне. Размещение кода, связанного с аутентификацией, на бизнес-уровне — это [подход Facebook](https://dev-blog.apollodata.com/graphql-at-facebook-by-dan-schafer-38d65ef075af).

![Бизнес-логика](images/business-logic.png)

Вы можете реализовать аутентификацию несколькими способами, что полностью соответствует вашим потребностям. Этот раздел покажет, как добавить аутентификацию по электронной почте в PinApp.

Аутентификация на основе электронной почты состоит из поля для ввода электронной почты. Как только пользователи укажут электронную почту, им придёт ссылка на приложение, в которой будет содержаться аутентификационный временный токен. Если пользователи перейдут в приложение с использованием действительного токена, то значит такому пользователю можно доверять (предполагаем, что почта пользователя подтверждена). А далее действительный токен будет заменен на токен аутентификации с более продолжительным сроком действия.

Самое большое преимущество подобной системы аутентификации, в отличие от старой доброй аутентификации по паролю, заключается в том, что вам вообще не нужно заниматься паролями. То, что мы не храним пароли также означает, что не требуется принимать крайних мер безопасности для их защиты. Еще это значит, что вашим пользователям не нужно заходить на очередной сайт, который просит их создать новый пароль.

```gherkin
Feature: Email based authentication

  Scenario: Send magic link
    Given a user enters his email address
    When he/she submits the form
    Then he/she should receive an email with a link to the app that contains his token

  Scenario: Verify email
    Given a user receives an email with a magic link
    When he/she clicks the link
    Then he/she should see a loading screen
    When the app verifies the token
    Then he/she should see an email confirmation message
```

Говоря в терминах GraphQL, оба эти действия (сценария) отражаются на мутациях в соответствующей GraphQL-схеме — `sendShortLivedToken` и `createLongLivedToken`. Первое действие получает в качестве аргумента электронное письмо типа `String`. Второе действие получает действительный токен, а возвращает новый токен.

```graphql
type Mutation {
  # ...
  sendShortLivedToken(email: String!): Boolean
  createLongLivedToken(token: String!): String
}
```

Склонируйте этот проект для того, чтобы следить за ходом внедрения аутентификации по электронной почте.

[Склонировать пример аутентификации по электронной почте](https://glitch.com/edit/#!/remix/pinapp-email-authentication)

Теперь давайте проанализируем, как выглядят резолверы, связанные с электронной почтой.

Резолвер `sendShortLivedToken` должен проверить, существует ли пользователь, соответствующий электронной почте. Если он не существует, он должен вставить его в базу данных. После этого он должен создать токен с небольшим сроком действия и отправить его на электронную почту пользователя.

```js
sendShortLivedToken: async (_, { email }) => {
  let user;
  const userExists = await database("users")
    .select()
    .where({ email });
  if (userExists.length) {
    user = userExists[0];
  } else {
    user = createUser(email);
    await database("users").insert(user);
  }
  const token = createShortLivedToken(user);
  return sendShortLivedToken(email, token);
};
```

Этот резолвер использует две функции из файла `business-logic.js` — `createShortLivedToken` и `sendShortLivedToken`.

Первый создает токен, используя функцию `jsonwebtoken` из NPM-пакета [`sign`](https://github.com/auth0/node-jsonwebtoken#jwtsignpayload-secretorprivatekey-options-callback). У этого токена срок действия ограничен пятью минутами.

```js
const createShortLivedToken = ({ email, id }) => {
  return jsonwebtoken.sign(
    { id, email },
    process.env.SECRET,
    {
      expiresIn: "5m"
    }
  );
};
```

Вторая функция `sendShortLivedToken` использует функцию, определенную в файле `email.js`, под названием `sendMail`.

```js
const sendShortLivedToken = (email, token) => {
  return sendMail({
    from: '"Julian" <julian@graphql.college>',
    to: email,
    text: `${process.env.APP_URL}/verify?token=${token}`,
    html: `<a href="${
      process.env.APP_URL
    }/verify?token=${token}" target="_blank">Authenticate</a>`,
    subject: "Auth token"
  });
};
```

Для отправки почты воспользуемся пакетом `nodemailer`. Эта библиотека позволяет отправлять электронные письма через SMTP-сервер. Самый простой способ создать почтовый сервер для разработки — [Ethereal](https://ethereal.email/messages. Это фиктивный почтовый сервис, разработанный создателями Nodemailer, является очень простым способом создания SMTP-службы для разработки. Разумеется, если вы хотите отправлять реальные электронные письма, вам следует использовать реальный сервис SMTP. [Sendgrid](https://sendgrid.com/) предлагает отличный бесплатный план.

```js
const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  host: "smtp.ethereal.email",
  port: 587,
  auth: {
    user: process.env.MAIL_USER,
    pass: process.env.MAIL_PASSWORD
  }
});

function sendMail({ from, to, subject, text, html }) {
  const mailOptions = {
    from,
    to,
    subject,
    text,
    html
  };
  return new Promise((resolve, reject) => {
    transporter.sendMail(mailOptions, (error, info) => {
      if (error) {
        return reject(error);
      }
      resolve(info);
    });
  });
}

module.exports = sendMail;
```

Реализация `createLongLivedToken` намного проще, чем `sendShortLivedToken`. Первый резолвер использует функцию из файла `business-logic.js`, проверяющая токен, переданный в качестве аргумента. Если токен действителен, функция создает токен с окончанием срока действия, равный тридцать дней.

```js
const createLongLivedToken = token => {
  try {
    const { id, email } = jsonwebtoken.verify(
      token,
      process.env.SECRET
    );
    const longLivedToken = jsonwebtoken.sign(
      { id, email },
      process.env.SECRET,
      { expiresIn: "30 days" }
    );
    return Promise.resolve(longLivedToken);
  } catch (error) {
    console.error(error);
    throw error;
  }
};
```

Двигайтесь дальше и сконфигурируйте склонированный проект в соответствии с вашей учетной записью Ethereal. Как только всё будет настроено, зайдите в GraphQL Playground, нажав кнопку «Show», и попробуйте авторизоваться, используя собственную электронную почту (или любую другую, так как Ethereal перехватывает все отправленные письма :D).

## 3.4 Организация файлов

В этом разделе вы узнаете, как использовать `graphql-import` для организации API-интерфейсов Node.js GraphQL по функциональным возможностям. Импорт GraphQL позволяет импортировать и экспортировать определения типов в GraphQL SDL.

На текущий момент все файлы в образце репозитория сгруппированы по ролям, но подобный не очень хорошо подходит для больших проектов. Сейчас все резолверы определены в файле `resolvers.js`, определения типов — в файле `schema.graphql`, а вся бизнес-логика — в файле b`business-logic.js`. По мере того, как проекты становятся все больше и больше, эти три файла станут слишком большими и неуправляемыми.

Текущая структура файла проекта выглядит таким образом:

![Текущая файловая структура](images/current-file-structure.png)

Вы собираетесь разделить `schema.graphql`, `resolvers.js` и `business-logic.js` на три функциональные возможности: `authentication`, `pins` и `search`. Окончательная структура каталогов будет такой, как показано ниже:

![Окончательная файловая структура](images/final-file-structure.png)

Склонируйте проект, если хотите посмотреть на окончательную версию.

[Клонировать пример организации файлов](https://glitch.com/edit/#!/remix/pinapp-files)

Основной точкой входа схемы GraphQL по-прежнему будет файл `schema.graphql`. Разница в том, что в нем не будут содержаться какие-либо определения типов, а вместо этого импортировать все типы из файла `schema.graphql` в каждой директории функциональной возможности. Основная схема будет импортировать остальные схемы, используя оператор `import`, предоставленный `graphql-import`. Его синтаксис: `# import * from "module-name.graphql"`.

```graphql
# import * from "authentication/schema.graphql"
# import * from "pins/schema.graphql"
# import * from "search/schema.graphql"
```

Такой способ импорта GraphQL SDL возможен, поскольку файл `schema.js` загружает `schema.graphql` следующим фрагментом кода:

```js
const { importSchema } = require("graphql-import");

const typeDefs = importSchema("schema.graphql");
// ...
```

Подобное схеме, главная точка входа для всех резолверов останется прежней. Это по-прежнему будет `resolvers.js`. Node.js уже предоставляет способ импорта и экспорта файлов с помощью `require`, поэтому вам не потребуется использовать дополнительную библиотеку. Даже если этот файл не нуждается в дополнительной библиотеке для импорта модулей, он использует библиотеку для объединения импортируемых резолверов. Можно писать на чистом JavaScript-коде, чтобы достичь аналогичного результата, однако `lodash.merge` — это отличный способ объединить множества JavaScript-объектов.

```js
const merge = require("lodash.merge");

const searchResolvers = require("./search/resolvers.js");
const authenticationResolvers = require("./authentication/resolvers.js");
const pinsResolvers = require("./pins/resolvers.js");

const resolvers = merge(
  searchResolvers,
  authenticationResolvers,
  pinsResolvers
);

module.exports = resolvers;
```

В завершение изменения файловой структуры, разделите `business-logic.js`, `resolvers.js` и `schema.graphql`. Разделите файл `business-logic.js` на `authentication/index.js` и `pins/index.js`. Разделите `resolvers.js` на`authentication/resolvers.js`, `pins/resolvers.js` и `search/resolvers.js`. И напоследок разделите `schema.graphql` на три новые директории.

Всё! Использование файловой структуры, основанной на функциональных возможностях — это масштабируемый способ организации кода. Такой подход может быть излишним для небольших проектов, но он с лихвой окупается в больших проектах.

## 3.5 Резюме

Вы узнали, как создать API на GraphQL, используя Apollo Server. Начав с простой GraphQL-схемы, вы узнали, как обернуть эту схему HTTP-слоем при помощи ApolloServer. Вы добавили слой базы данных, воспользовавшись Knex, а также аутентификацию по электронной почты с применением Nodemailer. Наконец, вы организовали свой проект по функциональным возможностям с использованием GraphQL Import.

В следующей главе вы научитесь, как создавать GraphQL-клиенты на примере Apollo Client, а также сделаете фронтенд на React, который взаимодействует с только что созданным API GraphQL.

