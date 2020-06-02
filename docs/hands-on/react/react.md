# Reactでサンプルアプリを構築する

作成したAppSyncにを利用するフロントエンドアプリケーションをReactで作成しましょう。

<!-- TODO: Reactに関する簡単な説明 -->
TODO: Reactに関する簡単な説明 Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged. It was popularised in the 1960s with the release of Letraset sheets containing Lorem Ipsum passages, and more recently with desktop publishing software like Aldus PageMaker including versions of Lorem Ipsum

## Create React Appを使ってセットアップする

Create React App（CRA）を使ってアプリケーションを生成します。任意の作業用ディレクトリに移動後、下記のコマンドを実行します。

```bash
npx create-react-app@3.4.1 --template typescript react-appsync-protected-by-auth0
cd react-appsync-protected-by-auth0
```

React RouterとAuth0のSPA用SDKをインストールします。

```bash
yarn add react-router-dom@5.1.2 @auth0/auth0-spa-js@1.8.1
yarn add -D @types/react-router-dom@5.1.2
```

Auth0をReactで利用する為のCustom Hookを作成します。

```typescript
// ./src/react-auth0-spa.tsx
import React from 'react';
import createAuth0Client, {
  Auth0Client, Auth0ClientOptions,
  getIdTokenClaimsOptions, GetTokenSilentlyOptions, GetTokenWithPopupOptions, IdToken, LogoutOptions,
  PopupConfigOptions,
  PopupLoginOptions, RedirectLoginOptions,
} from '@auth0/auth0-spa-js';

type Auth0ContextOptions = {
  isAuthenticated: boolean;
  user: any;
  loading: boolean;
  popupOpen: boolean;
  loginWithPopup: (options?: PopupLoginOptions, config?: PopupConfigOptions) => Promise<void>;
  handleRedirectCallback: (path?: string) => Promise<void>;
  getIdTokenClaims: (options?: getIdTokenClaimsOptions) => Promise<IdToken>;
  loginWithRedirect: (options?: RedirectLoginOptions) => Promise<void>;
  getTokenSilently: (options?: GetTokenSilentlyOptions) => Promise<any>;
  getTokenWithPopup: (options?: GetTokenWithPopupOptions, config?: PopupConfigOptions) => Promise<string>;
  logout: (options?: LogoutOptions) => void;
}

export type Auth0ProviderOptions = Auth0ClientOptions & {
  children: React.ReactElement;
  onRedirectCallback: Auth0ContextOptions['handleRedirectCallback'];
}

export const Auth0Context = React.createContext({} as Auth0ContextOptions);
export const useAuth0 = () => React.useContext<Auth0ContextOptions>(Auth0Context);
export const Auth0Provider: React.FC<Auth0ProviderOptions> = (
  {
    children,
    onRedirectCallback,
    ...initOptions
  }
) => {
  const [isAuthenticated, setIsAuthenticated] = React.useState<boolean>(false);
  const [user, setUser] = React.useState<any>(null);
  const [auth0Client, setAuth0] = React.useState<Auth0Client>();
  const [loading, setLoading] = React.useState<boolean>(true);
  const [popupOpen, setPopupOpen] = React.useState<boolean>(false);

  React.useEffect(() => {
    const initAuth0 = async () => {
      const auth0FromHook = await createAuth0Client(initOptions);
      setAuth0(auth0FromHook);

      if (window.location.search.includes('code=') &&
        window.location.search.includes('state=')) {
        const {appState} = await auth0FromHook.handleRedirectCallback();
        await onRedirectCallback(appState?.targetUrl);
      }

      const isAuthenticated = await auth0FromHook.isAuthenticated();

      setIsAuthenticated(isAuthenticated);

      if (isAuthenticated) {
        const user = await auth0FromHook.getUser();
        setUser(user);
      }

      setLoading(false);
    };
    initAuth0();
    // eslint-disable-next-line
  }, []);

  const loginWithPopup: Auth0ContextOptions['loginWithPopup'] = async (options, config) => {
    setPopupOpen(true);
    try {
      await auth0Client!.loginWithPopup(options, config);
    } catch (error) {
      console.error(error);
    } finally {
      setPopupOpen(false);
    }
    const user = await auth0Client!.getUser();
    setUser(user);
    setIsAuthenticated(true);
  };

  const handleRedirectCallback: Auth0ContextOptions['handleRedirectCallback'] = async (url) => {
    setLoading(true);
    await auth0Client!.handleRedirectCallback(url);
    const user = await auth0Client!.getUser();
    setLoading(false);
    setIsAuthenticated(true);
    setUser(user);
  };

  return (
    <Auth0Context.Provider
      value={{
        isAuthenticated,
        user,
        loading,
        popupOpen,
        loginWithPopup,
        handleRedirectCallback,
        getIdTokenClaims: (options) => auth0Client!.getIdTokenClaims(options),
        loginWithRedirect: (options) => auth0Client!.loginWithRedirect(options),
        getTokenSilently: (options) => auth0Client!.getTokenSilently(options),
        getTokenWithPopup: (options, config) => auth0Client!.getTokenWithPopup(options, config),
        logout: (options) => auth0Client!.logout(options)
      }}
    >
      {children}
    </Auth0Context.Provider>
  );
};
```

ログイン／ログアウト操作を行わせる為にNavBarを作成します。

```typescript
// ./src/components/NavBar.tsx
import React from "react";
import { useAuth0 } from "../react-auth0-spa";

export const NavBar = () => {
  const { isAuthenticated, loginWithRedirect, logout } = useAuth0();

  return (
    <div>
      {!isAuthenticated && (
        <button onClick={() => loginWithRedirect({})}>Log in</button>
      )}

      {isAuthenticated && <button onClick={() => logout()}>Log out</button>}
    </div>
  );
};

```

historyを生成し、どこからでもアクセスが出来るようにします。
TODO: この部分の説明が怪しい
useHistory() Hookを使わずに、createBrowserHistory()を使っているのは、前者はRouteコンポーネント配下でしか使えないが、後者はどこでも使える（少なくともpushの定義は出来る）からです。

```typescript
import { createBrowserHistory } from "history";
export const history = createBrowserHistory();
```

Auth0の設定をアプリケーションが取り込めるようにJSON形式で保存します。先程作成した、Auth0のApplication定義のSettingsタブからDomainとClient IDを転記してください。
今回はサンプルアプリケーションなので、環境（dev/stg/prd）差分を考慮せずパラメーターをハードコートしています。

![](img/auth0-get-domain-and-client-id.png)


```json
# ./src/auth-config.json
{
  "domain": "YOUR_DOMAIN",
  "clientId": "YOUR_CLIENT_ID"
}
```

作成したAuth0 Custom Hookをアプリケーションに結合させる為に、`index.tsx`を編集します。

```typescript
// ./src/index.tsx
import React from 'react';
import ReactDOM from 'react-dom';
import { App } from './App';
import * as serviceWorker from './serviceWorker';
import { history } from './utils/history';
import { Auth0Provider } from "./react-auth0-spa";
import config from "./auth_config.json";

const onRedirectCallback = async (url?: string) => {
  history.push(url ?? window.location.pathname);
};

ReactDOM.render(
  <React.StrictMode>
    <Auth0Provider
      domain={config.domain}
      client_id={config.clientId}
      redirect_uri={window.location.origin}
      onRedirectCallback={onRedirectCallback}
    >
      <App />
    </Auth0Provider>
  </React.StrictMode>,
  document.getElementById('root')
);

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: https://bit.ly/CRA-PWA
serviceWorker.unregister();

```

CRAで生成されたCSS等使わないので削除してしまいましょう。

- logo.svg
- index.tsx
- index.css
- App.css
- App.test.tsx

アプリケーションを実行して、ログインが出来るか確認します。

まず、テスト用のユーザーをAuth0に作成します。Auth0のDashbordを開き、**Users & Roles**、**Users**とメニューを選択し、**CREATE USER**を選択します。

![](img/auth0-list-users.png)

ユーザーのEmailとPasswordを入力し、**CREATE**を選択します。

![](img/auth0-create-user.png)

ベリファイのメールが届くのでリンクをクリックして、ユーザーを使用可能な状態に遷移させます。

![](img/auth0-verify-mail.png)

ユーザーの準備ができたので、アプリケーションを実行しログインが行えるか確認します。下記のコマンドで実行すると、デフォルトブラウザで [http://loclahost:3000](http://loclahost:3000) が開きます。自動でブラウザが開かない場合は手動でURLを入力してアクセスしてください。

```bash
yarn start
```

ブラウザで素朴な画面が表示されるので、Log inボタンを選択します。

![](img/app-login.png)

Auth0のユニバーサルログイン画面にリダイレクトされるので、テストユーザーのEmailとPasswordを入力し、**LOG IN**を選択します。

![](img/auth0-universal-login.png)

アプリケーションに対する認可を確認されるのでチェックアイコンを選択します。アプリケーションにアイコンを設定していないので、画像が表示されていませんね。

![](img/auth0-universal-login-authorize.png)

一瞬、**Loading...**と表示された後に、**Log out**ボタンだけの素朴な画面に戻れば成功です。

**Log out**を選択してログアウトしておきます。以降もアプリケーションを編集する前にはログアウトしてから行います。特に後ほど**audience**を設定する際には事前にログアウトが必須です。

![](img/app-logout.png)


## プロフィール画面にユーザー情報を表示する

**Token**の情報を表示するプロフィール画面を作成します。**Token**から情報を表示するのでログイン済みである必要があります。これに対応する為に、ログインしていなければAuth0のユニバーサルログイン画面へリダイレクトするPrivate Routeコンポーネントを作成します。

プロフィール画面を作成します。Auth0 Custom Hookからユーザー情報を取得し、表示します。

```typescript
// ./src/components/Profile.tsx
import React  from "react";
import { useAuth0 } from "../react-auth0-spa";

export const Profile = () => {
  const { loading, user } = useAuth0();

  if (loading || !user) {
    return <div>Loading...</div>;
  }

  return (
    <>
      <img src={user.picture} alt="Profile" />

      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <code>{JSON.stringify(user, null, 2)}</code>
    </>
  );
};
```

`NavBar.tsx`を変更し、Profile画面に移動できるようにします。

```tsx
// ./src/components/NavBar.tsx
import React from 'react';
import { useAuth0 } from '../react-auth0-spa';
import { Link } from 'react-router-dom';

export const NavBar = () => {
  const {isAuthenticated, loginWithRedirect, logout} = useAuth0();

  return (
    <div>
      {!isAuthenticated && (
        <button onClick={() => loginWithRedirect()}>Log in</button>
      )}
      {isAuthenticated && (
        <>
          <button onClick={() => logout()}>Log out</button>
          <span>
            <Link to="/">Home</Link> | <Link to="/profile">Profile</Link>
          </span>
        </>
      )}
    </div>
  );
};

```

`App.tsx`を変更しProfile画面へのルーティングを定義します。

```typescript
// ./src/App.tsx
import React from "react";
import { NavBar } from "./components/NavBar";
import { Router, Route, Switch } from "react-router-dom";
import { Profile } from "./components/Profile";
import { history } from "./utils/history";

export const App = () => {
  return (
    <div className="App">
      <Router history={history}>
        <header>
          <NavBar />
        </header>
        <Switch>
          <Route path="/" exact />
          <Route path="/profile" component={Profile} />
        </Switch>
      </Router>
    </div>
  );
};

```

まだ、リダイレクトが設定されていないので、下記のURLへアクセスするとLoading状態のまま遷移しません。

[http://localhost:3000/profile](http://localhost:3000/profile)

![](img/app-loading.png)

Private Route コンポーネントを作成します。このコンポーネントは React RouterのRouteコンポーネントのWrapperでログインしていなければ、ユニバーサルログイン画面にリダイレクトします。

```typescript
// ./src/components/PrivateRoute.tsx
import React from 'react';
import { Route, RouteProps } from 'react-router-dom';
import { useAuth0 } from '../react-auth0-spa';

export const PrivateRoute: React.FC<RouteProps> = ({ component: Component, path, ...rest }) => {
  const { loading, isAuthenticated, loginWithRedirect } = useAuth0();

  React.useEffect(() => {
    if (loading || isAuthenticated) {
      return;
    }
    const fn = async () => {
      await loginWithRedirect({
        appState: {targetUrl: window.location.pathname}
      });
    };
    fn();
  }, [loading, isAuthenticated, loginWithRedirect, path]);

  const render: RouteProps['render'] = props => {
    if (isAuthenticated && Component != null) {
      return <Component {...props} />;
    }

    return null;
  };

  return <Route path={path} render={render} {...rest} />;
};

```

`App.tsx`を変更して、Profile画面をPrivate Routeで保護します。

```typescript
import React from "react";
import { NavBar } from "./components/NavBar";
import { Router, Route, Switch } from "react-router-dom";
import { Profile } from "./components/Profile";
import { history } from "./utils/history";
import { PrivateRoute } from './components/PrivateRoute';

export const App = () => {
  return (
    <div className="App">
      <Router history={history}>
        <header>
          <NavBar />
        </header>
        <Switch>
          <Route path="/" exact />
          <PrivateRoute path="/profile" component={Profile} />
        </Switch>
      </Router>
    </div>

  );
};

```

アプリケーションを実行して、ログアウト状態でProfile画面にアクセスしようとするとリダイレクトされるか確認します。ログアウトした状態で下記のURLへアクセスします。

[http://localhost:3000/profile](http://localhost:3000/profile)

Auth0のユニバーサルログイン画面へリダイレクトされるので、EmailとPasswordを入力します。

Profile画面が表示されます。表示を確認できたら、ログアウトしておきます。

![](img/app-profile.png)