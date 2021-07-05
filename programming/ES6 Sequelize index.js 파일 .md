전 시리즈에서 Sequelize의 모델을 Class 문법으로 작성하는 방법에 대해 알아보았다.
이번 시리즈에선 Class 문법으로 작성한 모델 모듈을 `index.js` 파일에서 sequelize 객체에 import하는 방법에 대해 알아보려 한다.

아마 대부분의 분들은 아래처럼 `index.js`에서 model을 정의한 js파일을 불러올 것이다.

```javascript
const env = process.env.NODE_ENV || 'development';
const config = require(__dirname + '/../config/config.json')[env];

const db = {};

let sequelize;
if (config.use_env_variable) {
  sequelize = new Sequelize(process.env[config.use_env_variable], config);
} else {
  sequelize = new Sequelize(config.database, config.username, config.password, config);
}
db.sequelize = sequelize;
db.Sequelize = Sequelize;

db.Teacher = require('./teacher')(sequelize, Sequelize);
db.Student = require('./student')(sequelize, Sequelize);

module.exports = db;
```

해당 코드도 충분히 잘 돌아가지만, 새로운 모델 js 파일이 생성될 때마다 `index.js` 파일을 수정해줘야 한다는 문제점이 있다.
따라서, 모델 js파일을 자동으로 가져오는 `index.js` 파일을 수정한 코드는 아래와 같다.

```javascript
import fs from 'fs';
import path, { dirname } from 'path';

import { fileURLToPath } from 'url';
import configFile from '../config/config.js';

const config = configFile[env];

export const connection = new Sequelize(config.database, config.username, config.password, config);

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
const basename = path.basename(__filename);

fs.readdirSync(__dirname)
  .filter(file => file.indexOf('.') !== 0 && file !== basename && file.slice(-3) === '.js')
  .forEach(async file => {
    try {
      const module = await import(path.join(__dirname, file)); //1
      const model = module.default; //2
      model.initialize(connection); //3
    } catch (error) {
      console.log(error);
    }
  });
```

`import.meta` 객체는 현재 모듈에 대한 정보를 제공해준다. `import.meta.url`을 통해 얻은 현재 `index.js` 파일의 `url`을 `path`로 바꾸기 위해 `URL`모듈의 `fileURLToPath(url)` 함수를 사용한다.

```javascript
console.log(import.meta.url); // file:///Users/userFolder/Doc/projectApp/models/index.js
const __filename = fileURLToPath(import.meta.url);
console.log(__filename); // /Users/userFolder/Doc/projectApp/models/index.js
```

`index.js`의 파일이 있는 directory에 다른 모델 js 파일들을 읽기 위해 `fs`모듈의 `readdirSync` 함수를 사용한다.

```javascript
console.log(fs.readdirSync(__dirname)); // ['curriculum.js', 'index.js', 'student.js', 'teacher.js'];
```

directory에 있는 파일 중에 `index.js`파일과 js 파일이 아닌 파일들을 필터링한다.

그리고 `const module = await import(path.join(__dirname, file));`로 해당 모델 js 파일을 동적으로 `import`한다. 그리고 우린 위에서 모델 Class에 `default export`를 사용하였다. `default export`한 모듈을 사용하려면 모듈 객체의 `default` 프로퍼티를 사용하면 된다. (출처 : [동적으로 모듈 가져오기](https://ko.javascript.info/modules-dynamic-imports))

```javascript
// 주석이 있는 줄을 위 코드처럼 두 줄로 줄여서 쓸 수 있다.
const { default: model } = await import(path.join(__dirname, file));
model.initialize(connection);
```

이제 `app.js`에서 `index.js`파일을 `import`하고 실행해보자.

```javascript
...
import { connection } from './models/index.js';
connection.sync({});
...
```

```bash
❯ npm start

> projectApp@0.0.0 start /Users/userFolder/Doc/projectApp
> node app.js

(node:13497) ExperimentalWarning: The ESM module loader is experimental.
initialize Model Class
[ 'curriculum.js', 'index.js', 'student.js', 'teacher.js' ]
Load model associations
3000 번 포트에서 대기중
```

여기서 문제점이 발생하였다. `app.js`를 실행해도 데이터베이스에 새로 추가한 모델들이 생성이 안된 것이다.
sequelize의 문제는 없었는데 왜 안되는지 알아보기 위해 `app.js`의 sequelize의 객체인 `connection` 을 콘솔로 확인하였다.

```json
// console.log(connection.models);
{}
```

확인해보니, sequelize 객체에 새로 추가한 모델들이 초기화가 안되어 있었다. 이유를 한동안 찾아보니 원인을 알게 되었다. <br>
원인은 `index.js` 파일에서 모델 파일들을 불러오는 `import` 함수와 `forEach` 함수의 문제였다.

> `import(module)` 표현식은 모듈을 읽고 이 모듈이 내보내는 것들을 모두 포함하는 객체를 담은 이행된 프라미스를 반환합니다.(출처 : [동적으로 모듈 가져오기](https://ko.javascript.info/modules-dynamic-imports))

import 함수는 프라미스를 반환하기 때문에 `async/await` 표현식을 사용해야 하는데 `forEach` 내 익명 함수는 동기식이기 때문이다. 따라서, 모델 파일을 읽어와 `array`로 만들고 비동기적으로 `import`하는 코드로 수정했다.

```javascript
// 비동기식으로 모델 파일 import 함수
async function importModel(file) {
  const { default: model } = await import(path.join(__dirname, file));
  model.initialize(connection, Sequelize);
}

// 각 비동기 처리를 위한 Promise를 반환하는 function을 map 메서드로 묶어 Promise 배열을 반환하고 이를 Promise.all을 통해서 한 번에 처리하는 함수
async function processArray(array) {
  const promises = array.map(importModel);
  await Promise.all(promises);
  console.log('Done!');
}

console.log('initialize Model Class');

const files = fs.readdirSync(__dirname).filter(file => file.indexOf('.') !== 0 && file !== basename && file.slice(-3) === '.js');

processArray(files);
```

하지만 이 방식 역시 문제점이 존재하였다. <br>
그건 바로 `index.js`가 import되는 `app.js`에서 `index.js` 모듈을 비동기식으로 import한다는 것이었다.

```bash
❯ npm start

> projectApp@0.0.0 start /Users/userFolder/Doc/projectApp
> node app.js

(node:22410) ExperimentalWarning: The ESM module loader is experimental.
initialize Model Class
{} // console.log(connection.model);
3000 번 포트에서 대기중
Done!  // sequelize 객체를 import한 후 모델 js 파일들이 초기화되었다.
```

해당 문제를 해결하기 위해 `babel`을 사용하기로 했다. CommonJS의 경우 기본적으로 모듈을 import할 때 동기로 동작한다. Node.js는 CommonJS가 표준이기 때문에 ES6를 CommonJS로 변환해주는 bebel과 같은 번들러가 필요하다. (참고자료 : [You don't know JS module](https://ui.toast.com/weekly-pick/ko_20190418))

> ES6(ES2105) 이상의 최신 자바스크립트 문법으로 작성된 코드가 노드JS(NodeJS)에서 실행이 안 되는 경우가 종종 있다. 이럴 경우 어쩔 수 없이 예전 자바스크립트 문법으로 코드를 재작성하기도 한다.
> (출처 : [babel 설치 및 Node.js로 ES6 코드 실행하기](https://www.daleseo.com/js-babel-node/))

링크에서 설명한대로 `package.json`을 수정하고 `app.js`파일과 `index.js`파일을 다시 수정하였다.

```json
// package.json
...
"type": "commonjs",
  "scripts": {
    "start": "babel-node app.js",
    "build": "babel ./app.js -w -d dist/js"
  },
...
```

```json
// .babelrc
{
  "presets": ["@babel/env"],
  "plugins": ["@babel/plugin-proposal-class-properties"]
}
```

```javascript
// app.js
import { connection } from './models/index.js';
console.log(connection.models);
connection.sync({ force: true });
```

```javascript
// models/index.js
import Sequelize from 'sequelize';
import fs from 'fs';
import path from 'path';
import configFile from '../config/config.js';

const env = process.env.stage === 'dev' ? 'development' : process.env.stage === 'test' ? 'test' : 'production';
const config = configFile[env];

export const connection = new Sequelize(config.database, config.username, config.password, config);

const basename = path.basename(__filename);

console.log('initialize Model Class');
fs.readdirSync(__dirname)
  .filter(file => file.indexOf('.') !== 0 && file !== basename && file.slice(-3) === '.js')
  .forEach(file => {
    const { default: model } = require(path.join(__dirname, file));
    model.initialize(connection, Sequelize);
  });
```

`package.json`에 정의한 `npm start`로 `app.js`를 실행해보면

```bash
❯ npm start

> projectApp@0.0.0 start /Users/userFolder/Doc/projectApp
> babel-node app.js

initialize Model Class
{ Curriculum: Curriculum, Student: Student, Teacher: Teacher }
3000 번 포트에서 대기중
Executing (default): DROP TABLE IF EXISTS `Teachers`;
Executing (default): DROP TABLE IF EXISTS `Students`;
Executing (default): DROP TABLE IF EXISTS `Curriculums`;
Executing (default): DROP TABLE IF EXISTS `Curriculums`;
Executing (default): CREATE TABLE IF NOT EXISTS `Curriculums` (`id` INTEGER NOT NULL auto_increment , `title` VARCHAR(255) NOT NULL, `numberOfTimes` INTEGER NOT NULL, `price` INTEGER NOT NULL, `canExhibit` TINYINT(1) NOT NULL, `target` VARCHAR(255), `textbook` VARCHAR(255) NOT NULL, `totalDescription` TEXT, `place` VARCHAR(255) NOT NULL, `isUntact` TINYINT(1) NOT NULL, `createdAt` DATETIME NOT NULL, `updatedAt` DATETIME NOT NULL, `deletedAt` DATETIME, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_unicode_ci;
Executing (default): SHOW INDEX FROM `Curriculums` FROM `mvcTest`
Executing (default): DROP TABLE IF EXISTS `Students`;
Executing (default): CREATE TABLE IF NOT EXISTS `Students` (`id` CHAR(36) NOT NULL , `nickname` VARCHAR(30) NOT NULL, `phoneNumber` VARCHAR(255), `gender` CHAR(1), `certifiedPhoneNumber` TINYINT(1) NOT NULL DEFAULT false, `isActive` TINYINT(1) DEFAULT true, `profileImg` VARCHAR(255) DEFAULT '', `createdAt` DATETIME NOT NULL, `updatedAt` DATETIME NOT NULL, `deletedAt` DATETIME, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_unicode_ci;
Executing (default): SHOW INDEX FROM `Students` FROM `mvcTest`
Executing (default): DROP TABLE IF EXISTS `Teachers`;
Executing (default): CREATE TABLE IF NOT EXISTS `Teachers` (`id` CHAR(36) NOT NULL , `phoneNumber` VARCHAR(255), `nickname` VARCHAR(30) NOT NULL, `fcmToken` VARCHAR(1000), `birthday` DATE, `gender` CHAR(1), `introduction` TEXT, `recommendNum` INTEGER DEFAULT 0, `canUntact` TINYINT(1), `canRentalInstrument` TINYINT(1), `certificatedEdu` TINYINT(1) DEFAULT false, `certifiedPhoneNumber` TINYINT(1) NOT NULL DEFAULT false, `isActive` TINYINT(1) DEFAULT true, `profileImg` VARCHAR(255) DEFAULT '', `createdAt` DATETIME NOT NULL, `updatedAt` DATETIME NOT NULL, `deletedAt` DATETIME, PRIMARY KEY (`id`)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_unicode_ci;
Executing (default): SHOW INDEX FROM `Teachers` FROM `mvcTest`
```

sequelize의 로깅을 끄고 싶은 분은 `config.js` 파일에서 `logging:false` 옵션을 추가하면 된다.

```javascript
...
port: 3306,
host: env.DB_Host,
username: env.DB_Username,
password: env.DB_Password,
dialect: 'mysql',
timezone: '+09:00',
logging: false,
dialectOptions: {
  charset: 'utf8mb4',
  dateStrings: true,
  typeCast: true
},
...
```

정의했던 모델들이 제대로 생성되는 것을 볼 수 있다.
![모델](./스크린샷%202021-06-22%20오후%209.33.36.png)

앞 시리즈에서 정의한 `Student` 모델 역시 데이터베이스에 컬럼이름, 데이터타입, 옵션 모두 문제없이 등록되어 있다.
![스키마](./스크린샷%202021-06-22%20오후%209.35.43.png)

지금까지 모델 모듈을 `index.js` 파일에서 sequelize 객체에 import하는 방법에 대해 알아보았다.
개발 과정에서 ES6의 모듈 동기 처리에서 내가 아직 ES6 자체에서 제공하는 동기 방식을 모르고 JS 코딩 기법이 부족하여 babel을 사용하게 되었다.
(혹시 babel을 사용하지 않고도 처리할 수 있는 방법을 아시는 분은 답변을 주시면 감사하겠습니다. 🙇)

다음은 생성한 모델끼리 관계형을 설정하고 모델별 커스텀 메서드를 생성하는 방법에 대해 알아보자.

다음 시리즈에서 계속...
