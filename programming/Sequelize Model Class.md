프로젝트를 진행하며 백엔드로 Node.js의 Express 프레임워크를 사용하던 중 ES6 문법으로 개발을 진행하였다.

또한 데이터베이스로 MySQL을 사용하기 때문에 Node.js의 ORM 라이브러리인 Sequelize를 사용하였다. 그러던 중 Sequelize를 ES6 문법의 module, Class 등을 사용하며 겪은 삽질을 간단하게 정리하였다.

###### 개발 환경

- node : 12.18.4
- babel/cli : 7.14.3
- express : 4.16.1
- Sequelize : 6.2.0 (Sequelize CLI [Node: 12.18.4, CLI: 6.2.0, ORM: 6.6.2])

프로젝트의 대략적인 directory 구조는 아래와 같다.

```bash
~/Doc/projectApp ❯ tree -L 2
.
├── README.md
├── app.js
├── config
│   └── config.js
├── controllers
├── jobs
├── libs
├── loaders
├── migrations
├── models
│   ├── curriculum.js
│   ├── index.js
│   ├── student.js
│   └── teacher.js
├── node_modules
├── package-lock.json
├── package.json
├── public
├── seeders
├── services
└── subscribers

12 directories, 4 files
```

아마 대부분의 분들은 Sequelize로 모델을 정의할 때 아래의 코드로 js 파일을 생성할 것이다.

```javascript
// student.js
const Sequelize = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  return sequelize.define(
    'student',
    {
      id: {
        type: Sequelize.CHAR(36),
        defaultValue: Sequelize.UUIDV1,
        primaryKey: true,
        allowNull: false,
      },
      nickname: {
        type: Sequelize.STRING(30),
        allowNull: false,
      },
      phoneNumber: {
        type: Sequelize.STRING,
        allowNull: true,
      },
      gender: {
        type: Sequelize.CHAR(1),
        allowNull: true,
      },
      certifiedPhoneNumber: {
        type: Sequelize.BOOLEAN,
        defaultValue: false,
        allowNull: false,
      },
      isActive: {
        type: Sequelize.BOOLEAN,
        defaultValue: true,
      },
      profileImg: {
        type: Sequelize.STRING,
        defaultValue: '',
      },
    },
    {
      timestamps: true,
      paranoid: true,
      charset: 'utf8mb4',
      collate: 'utf8mb4_unicode_ci',
    }
  );
};
```

위 코드로도 충분히 Sequelize의 define method를 통해 테이블을 생성할 수 있다. 하지만, 이 글의 목적처럼 위 js 파일을 Class와 ES6 문법으로 재구성하면 아래와 같은 코드가 된다.

```javascript
import pkg from 'sequelize';
const { Model } = pkg;

export default class Student extends Model {
  static initialize(sequelize, DataTypes) {
    super.init(
      {
        id: {
          type: DataTypes.CHAR(36),
          defaultValue: DataTypes.UUIDV1,
          primaryKey: true,
          allowNull: false,
        },
        nickname: {
          type: DataTypes.STRING(30),
          allowNull: false,
        },
        phoneNumber: {
          type: DataTypes.STRING,
          allowNull: true,
        },
        gender: {
          type: DataTypes.CHAR(1),
          allowNull: true,
        },
        certifiedPhoneNumber: {
          type: DataTypes.BOOLEAN,
          defaultValue: false,
          allowNull: false,
        },
        isActive: {
          type: DataTypes.BOOLEAN,
          defaultValue: true,
        },
        profileImg: {
          type: DataTypes.STRING,
          defaultValue: '',
        },
      },
      {
        sequelize,
        timestamps: true,
        paranoid: true,
        charset: 'utf8mb4',
        collate: 'utf8mb4_unicode_ci',
      }
    );
  }
}
```

sequelize에서 Model과 DataTypes를 바로 import하면 에러가 발생한다.

```bash
import { Model, DataTypes } from 'sequelize';
         ^^^^^
SyntaxError: The requested module 'sequelize' is expected to be of type CommonJS, which does not support named exports. CommonJS modules can be imported by importing the default export.
For example:
import pkg from 'sequelize';
const { Model, DataTypes } = pkg;
    at ModuleJob._instantiate (internal/modules/esm/module_job.js:97:21)
    at async ModuleJob.run (internal/modules/esm/module_job.js:136:20)
    at async Loader.import (internal/modules/esm/loader.js:179:24)
    at async file:///Users/jiwonjeoung/Documents/tuningApp/models/index.js:38:19
(node:31866) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). To terminate the node process on unhandled promise rejection, use the CLI flag `--unhandled-rejections=strict` (see https://nodejs.org/api/cli.html#cli_unhandled_rejections_mode). (rejection id: 3)
```

에러 메시지를 확인해보니 sequelize의 경우 CommonJS 모듈을 사용하는데, ES6 문법인 import를 사용하기 위한 `package.json`의 type을 module로 한 것에 충돌이 일어난 것으로 보인다. 따라서, 에러 메시지에 친절히 써져 있는대로 `import pkg from 'sequelize'; const { Model, DataTypes } = pkg;`로 수정하면 된다.
(참고자료 : [You don't know JS module](https://ui.toast.com/weekly-pick/ko_20190418))

Student 라는 클래스를 생성하고 이 클래스는 Sequelize의 Model 클래스를 상속 받는다. Sequelize의 Model 파일을 보면 typescript로 작성된 코드와 주석으로 된 설명을 볼 수 있다. 그 중 init 메서드의 설명 부분을 번역하였다.

````typescript
  /**
   * 속성 및 옵션을 사용하여 DB의 테이블을 나타내는 모델을 초기화합니다.
   *
   * 테이블 컬럼은 두 번째 인수로 제공되는 해시로 정의됩니다. 해시의 각 속성은 컬럼을 나타냅니다. 간단한 테이블 정의는 다음과 같습니다. :
   *
   * ```js
   * Project.init({
   *   columnA: {
   *     type: Sequelize.BOOLEAN,
   *     validate: {
   *       is: ['[a-z]','i'],        // will only allow letters
   *       max: 23,                  // only allow values <= 23
   *       isIn: {
   *         args: [['en', 'zh']],
   *         msg: "Must be English or Chinese"
   *       }
   *     },
   *     field: 'column_a'
   *     // Other attributes here
   *   },
   *   columnB: Sequelize.STRING,
   *   columnC: 'MY VERY OWN COLUMN TYPE'
   * }, {sequelize})
   *
   * sequelize.models.modelName // 생성한 모델은 정의된 클래스 이름으로 sequelize의 객체의 models 속성에서 사용할 수 있습니다.
   * ```
   *
   * 위에 표시된 대로 컬럼 정의는 문자열, Sequelize 생성자에 사전 정의된 데이터타입 중 하나에 대한 참조이거나 컬럼타입 및 default value 및 외래키 제약식, 커스텀 setters와 getters 등과 같은 기타 속성을 지정할 수 있는 객체 일 수 있습니다.
   * @param attributes
   *  각 속성이 테이블의 컬럼인 객체입니다. 각 열은 DataType, 문자열 또는 유형 설명 객체 일 수 있으며 아래에 설명 된 속성이 있습니다. :
   * @param options 이 옵션은 Sequelize 생성자에 제공된 기본 정의 옵션과 병합됩니다.
   * @return 초기화된 모델을 리턴합니다.
   */
  public static init<M extends Model>(
    this: ModelStatic<M>,
    attributes: ModelAttributes<M, M['_attributes']>, options: InitOptions<M>
  ): Model;
````

다시 Student 클래스 모듈을 보면 initialize 메서드에 static 키워드가 있어 해당 메서드를 정적 메서드로 정의한 걸 볼 수 있다.

> 정적 메서드는 클래스의 인스턴스 없이 호출이 가능하며 클래스가 인스턴스화되면 호출할 수 없다. 정적 메서드는 종종 어플리케이션의 유틸리티 함수를 만드는데 사용된다.
> (출처 : https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Classes/static)

즉, 우리가 어플리케이션에서 사용자를 생성할 때 그들의 컬럼의 값들은 변하지만 컬럼과 Student의 관계나 커스텀 메서드 등은 변하지 않기 때문에 static 메서드를 사용한 것이다.

또한 Constructor(생성자) 메서드를 사용하지 않았는데 이 역시 위 static 키워드를 사용한 것과 비슷한 맥락이다. constructor 메서드는 class 로 생성된 객체를 생성하고 초기화하기 위한 특수한 메서드이다. 하지만 우린 Student 클래스로 생성된 객체를 초기화하는 메서드를 사용할 필요가 없다. 이유는 Student 클래스는 Sequelize에서 database의 Students 테이블을 ORM으로 사용하기 위한 하나의 유틸리티 클래스이기 때문이다.

> 유틸리티 클래스 <br> - 인스턴스 메서드와 인스터스 변수를 일절 제공하지 않고, 정적 메서드와 변수만을 제공하는 클래스를 뜻한다. <br> - 클래스 본래의 목적인 '데이터와 데이터 처리를 위한 로직의 캡슐화'를 실행하는 것이 아닌,'비슷한 기능의 메서드와 상수를 모아서 캡슐화'한 것이 유틸리티 클래스이다.
> (출처 : https://morningcoding.tistory.com/entry/Java15-%ED%81%B4%EB%9E%98%EC%8A%A4-%EB%B3%80%EC%88%98-%ED%81%B4%EB%9E%98%EC%8A%A4-%EB%A9%94%EC%84%9C%EB%93%9C%EC%99%80-%EC%9C%A0%ED%8B%B8%EB%A6%AC%ED%8B%B0-%ED%81%B4%EB%9E%98%EC%8A%A4)

따라서, Student 클래스는 database의 student 테이블에 데이터를 생성하는 정적 메서드를 따로 작성해야 한다.

```javascript
  static async create(args) {
    return await this.create(args);
  }
```

Class Model의 CRUD에 대해선 나중에 보기로 하고 이제 테이블 Class Model을 데이터베이스와 연결하고 생성하는 방법에 대해 알아보자.

다음 시리즈에서 계속...
