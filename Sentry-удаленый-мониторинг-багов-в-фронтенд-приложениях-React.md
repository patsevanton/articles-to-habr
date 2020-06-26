Мы изучаем использование Sentry с React.

![](https://miro.medium.com/max/1280/1*xZyTGkgbJw59NNl5UL5gQA.jpeg)

Эта статья является частью серии, начинающейся с сообщения об ошибках Sentry на примере: [Часть 1](https://medium.com/@johntucker_48673/sentry-error-reporting-by-example-part-1-999b2df11556). 

**Реализация React**  

Сначала нам нужно добавить новый проект Sentry для этого приложения; с сайта Sentry. В этом случае мы выбираем React.

Мы вновь реализуем наши две кнопки, Hello и Error, приложение с React. Мы начинаем с создания нашего стартового приложения:

```
npx create-react-app react-app
```

Затем мы импортируем пакет Sentry:

```
yarn add @sentry/browser
```

и инициализируем его:

react-app / src / index.js

```javascript
...
import * as Sentry from '@sentry/browser';

const RELEASE = '0.1.0';
if (process.env.NODE_ENV === 'production') {
  Sentry.init({
    dsn: 'https://303c04eac89844b5bfc908ceffc6757c@sentry.io/1289887',
    release: RELEASE,
  });
}
...
```

Наблюдения:

- Во время разработки у нас есть другие механизмы для наблюдения за проблемами, например консоль, поэтому мы включаем Sentry только для производственных сборок


Затем мы реализуем наши кнопки Hello и Error и добавляем их в приложение:

*react-app / src / Hello.js*

```javascript
import React, { Component } from 'react';
import * as Sentry from '@sentry/browser';

export default class Hello extends Component {
  state = {
    text: '',
  };
  render() {
    const { text } = this.state;
    return (
      <div>
        <button
          onClick={this.handleClick}
        >
          Hello
        </button>
        <div>{text}</div>
      </div>
    )
  }

  handleClick = () => {
    this.setState({
      text: 'Hello World',
    });
    try {
      throw new Error('Caught');
    } catch (err) {
      if (process.env.NODE_ENV !== 'production') {
        return;
      }
      Sentry.captureException(err);
    }
  }
}
```

*react-app / src / MyError.js*

```
import React, { Component } from 'react';

export default class MyError extends Component {
  render() {
    return (
      <div>
        <button
          onClick={this.handleClick}
        >
          Error
        </button>
      </div>
    )
  }

  handleClick = () => {
    throw new Error('Uncaught');
  }
}
```

*react-app / src / App.js*

```
...
import Hello from './Hello';
import MyError from './MyError';

class App extends Component {
  render() {
    return (
      <div className="App">
        ...
        <Hello />
        <MyError />
      </div>
    );
  }
}

export default App;
```

**Проблема (Исходные Карты)**

Мы можем протестировать Sentry с производственной сборкой, введя:

```
yarn build
```

и из build папки введите:

```
npx http-server -c-1
```

Проблема, с которой мы немедленно столкнемся, заключается в том, что записи об ошибке Sentry ссылаются на номера строк в уменьшенном пакете; не очень полезно.

![](https://miro.medium.com/max/1104/1*7W-jTpF1uRPDutjDYCOzyA.png)

Служба Sentry объясняет это, вытягивая исходные карты для уменьшенного пакета после получения ошибки. В этом случае мы бежим от localhost (недоступного службой Sentry).

**Решения (Исходные Карты)**

Решение этой проблемы сводится к запуску приложения с общедоступного веб-сервера. Одна простая кнопка ответа на него, чтобы использовать сервис GitHub Pages (бесплатно). Шаги для использования обычно следующие:

1. Скопируйте содержимое папки *build* в папку *docs* в корневом каталоге репозитория.

2. Включите *GitHub Pages* в репозитории (из GitHub), чтобы использовать папку docs в *master* ветви

3. Перенесите изменения на GitHub


**Примечание**: после того, как я понял, что мне нужно использовать *create-create-app* функция домашней страницы для запуска приложения. Сводилось к добавлению следующего к package.json:

```
"homepage": "https://larkintuckerllc.github.io/hello-sentry/"
```

Окончательная версия запущенного приложения доступна по адресу:

https://larkintuckerllc.github.io/hello-sentry/

**Иллюстрация Пойманных Ошибок**

Давайте пройдем через нажатие кнопки Hello.

![](https://miro.medium.com/max/1280/1*u21flUaBBFrt0jvm-4Q2vg.png)

С ошибкой, появляющейся следующим образом:

![](https://miro.medium.com/max/1062/1*XpsJF6CrhxXHFw9QXDZzZQ.png)

Наблюдения:

- Этот отчет об ошибке не может быть более ясным, **BRAVO**.


**Иллюстрация Неучтенных Ошибок**

Аналогично, давайте пройдем через нажатие кнопки *Error*.

![](https://miro.medium.com/max/1280/1*KFEoQyQmktx6uKJ-_K1v1g.png)

С ошибкой, появляющейся следующим образом:

![](https://miro.medium.com/max/1082/1*GMHbHr9Xu1JxoTl458b_ZQ.png)

**Лучшая обработка неучтенных ошибок (рендеринг)**

> Введение Границ Ошибок
>
> Ошибка JavaScript в части пользовательского интерфейса не должна нарушать работу всего приложения. Чтобы решить эту проблему для пользователей React, React 16 вводит новое понятие "границы ошибок".
>
> Границы ошибок – это компоненты React, которые ловят ошибки JavaScript в любом месте своего дочернего дерева компонентов, регистрируют эти ошибки и отображают резервный пользовательский интерфейс вместо дерева компонентов, которое разбилось. Границы ошибок улавливают ошибки во время рендеринга, в методах жизненного цикла и в конструкторах всего дерева под ними.
>
> …
>
> Новое поведение для необнаруженных ошибок
>
> Это изменение имеет важное значение. Начиная с React 16, ошибки, которые не были пойманы какой-либо границей ошибок, приведут к размонтированию всего дерева компонентов React.
>
> — *Dan Abramov —* [*Error Handling in React 16*](https://reactjs.org/blog/2017/07/26/error-handling-in-react-16.html)

Важное уточнение, которое заняло у меня некоторое время, прежде чем я понял это, заключается в том, что **вышеупомянутое поведение работает только с ошибками, генерируемыми в методе рендеринга (или, что более вероятно, в любом из методов жизненного цикла)**. Например, использование границ ошибок не принесло бы никакой пользы с нашей кнопкой *Error*; эта ошибка была в обработчике щелчка.

Давайте создадим пример ошибки визуализации, а затем используем границы ошибок для более изящной обработки ошибки.

*react-app / src / MyRenderError*

```
import React, { Component } from 'react';

export default class MyRenderError extends Component {
  state = {
    flag: false,
  };
  render() {
    const { flag } = this.state;
    return (
      <div>
        <button
          onClick={this.handleClick}
        >
          Render Error
        </button>
        { flag && <div>{flag.busted.bogus}</div> }
      </div>
    )
  }

  handleClick = () => {
    this.setState({
      flag: true,
    });
  }
}
```

Наблюдение:

- При нажатии кнопки, *React* будет отображаться *flag.busted.bogus*, которая порождает ошибку

- Без границы ошибки все дерево компонентов будет размонтировано


Затем мы пишем наш код границы ошибки (использует новый метод жизненного цикла *componentDidCatch*); это, по сути, пример, приведенный в статье Дэна Абрамова:

*react-app / src / ErrorBoundary.js*

```
import React, { Component } from 'react';
import * as Sentry from '@sentry/browser';

export default class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  componentDidCatch(err, info) {
    this.setState({ hasError: true });
    Sentry.captureException(err);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}
```

Наконец, мы используем этот компонент:

*react-app / src / App.js*

```
...
import MyRenderError from './MyRenderError';

class App extends Component {
  render() {
    return (
      <ErrorBoundary>
        <div className="App">
          ...
        </div>
      </ErrorBoundary>
    );
  }
}
...
```

При этом нажатие кнопки Render Error отображает резервный пользовательский интерфейс и сообщает об ошибке Sentry.

![](https://miro.medium.com/max/1280/1*eIPr0P5kvppPx4tLJC0W1A.png)

![](https://miro.medium.com/max/1280/1*JU-Y-1a3WptQg0WrN13TYg.png)

**Завершение**

Надеюсь, вам было это полезно.

P.S. Телеграм чат по Sentry https://t.me/sentry_ru