---
title: Add Login
description: This tutorial demonstrates how to add user login functionality to a React application using Auth0.
budicon: 448
topics:
  - quickstarts
  - spa
  - react
  - login
github:
  path: 01-Login
contentType: tutorial
useCase: quickstart
---
<!-- markdownlint-disable MD002 -->

::: note
**New to Auth?** Learn [How Auth0 works](/overview), how it [integrates with Single-Page Applications](/architecture-scenarios/application/spa-api) and which [protocol](/flows/concepts/implicit) it uses.
:::

## Setup

### Install Auth0 SPA JS SDK

First you will need to install the [Auth0 Client SDK](https://github.com/auth0/auth0-spa-js).

```bash
# Using NPM
npm install @auth0/auth0-spa-js

# Using Yarn
yarn add @auth0/auth0-spa-js
```

### Configure Application Keys
In order for the SDK to use the correct application you will need to configure the correct application keys. Create a new file `src/authConfig.json` in your `src` folder and copy the following code block into it

::: note
You can locate your `domain` and `client id` from the Dashboard [Applications](${manage_url}/#/applications) page and navigating into the settings for the desired application.
:::

```json
// src/authConfig.json
{
  "domain": "${account.namespace}",
  "clientId": "${account.clientId}"
}
```

### Configure Callback URLs

A callback URL is a URL in your application where Auth0 redirects the user after they have authenticated. 

The callback URL for your app must be whitelisted in the **Allowed Callback URLs** field in your [Application Settings](${manage_url}/#/applications/${account.clientId}/settings). If this field is not set, users will be unable to log in to the application and will get an error.

::: note
If you are testing your application locally, you should set the **Allowed Callback URL** to `http://localhost:3000/callback`
:::

### Configure Allowed Web Origins

You need to whitelist the URL for your app in the **Allowed Web Origins** field in your [Application Settings](${manage_url}/#/applications/${account.clientId}/settings). If you don't whitelist your application URL, the application will be unable to automatically refresh the authentication tokens and your users will be logged out the next time they visit the application, or refresh the page.

::: note
If you are testing your application locally,  you should set the **Allowed Web Origins** to `http://localhost:3000/`
:::

### Configure Logout URLs

A logout URL is a URL in your application that Auth0 can return to after the user has been logged out of the authorization server. This is specified in the `returnTo` query parameter.

The logout URL for your app must be whitelisted in the **Allowed Logout URLs** field in your [Application Settings](${manage_url}/#/applications/${account.clientId}/settings). If this field is not set, users will be unable to log out from the application and will get an error.

::: note
If you are testing your application locally, the logout URL you need to whitelist in the **Allowed Logout URLs** field is `http://localhost:3000`.
:::

### Auth0 React Wrapper

In order to work with the Auth0 SDK in a more idiomatic way, we have put together a set of custom [React hooks](https://reactjs.org/docs/hooks-intro.html).

Create a new file `auth0-react-spa.js` in your `src` folder and copy the following code into it

```js
// src/auth0-react-spa.js
import React, { useState, useEffect, useContext } from "react";
import createAuth0Client from "@auth0/auth0-spa-js";

const DEFAULT_REDIRECT_CALLBACK = () =>
  window.history.replaceState({}, document.title, window.location.pathname);

export const Auth0Context = React.createContext();
export const useAuth0 = () => useContext(Auth0Context);
export const Auth0Provider = ({
  children,
  onRedirectCallback = DEFAULT_REDIRECT_CALLBACK,
  ...initOptions
}) => {
  const [isAuthenticated, setIsAuthenticated] = useState();
  const [user, setUser] = useState();
  const [auth0Client, setAuth0] = useState();
  const [loading, setLoading] = useState(true);
  const [popupOpen, setPopupOpen] = useState(false);

  useEffect(() => {
    const initAuth0 = async () => {
      const auth0FromHook = await createAuth0Client(initOptions);
      setAuth0(auth0FromHook);

      if (window.location.search.includes("code=")) {
        const { appState } = await auth0FromHook.handleRedirectCallback();
        onRedirectCallback(appState);
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

  const loginWithPopup = async (params = {}) => {
    setPopupOpen(true);
    try {
      await auth0Client.loginWithPopup(params);
    } catch (error) {
      console.error(error);
    } finally {
      setPopupOpen(false);
    }
    const user = await auth0Client.getUser();
    setUser(user);
    setIsAuthenticated(true);
  };

  const handleRedirectCallback = async () => {
    setLoading(true);
    await auth0Client.handleRedirectCallback();
    const user = await auth0Client.getUser();
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
        getIdTokenClaims: (...p) => auth0Client.getIdTokenClaims(...p),
        loginWithRedirect: (...p) => auth0Client.loginWithRedirect(...p),
        getTokenSilently: (...p) => auth0Client.getTokenSilently(...p),
        getTokenWithPopup: (...p) => auth0Client.getTokenWithPopup(...p),
        logout: (...p) => auth0Client.logout(...p)
      }}
    >
      {children}
    </Auth0Context.Provider>
  );
};
```

### Provide Context to your Application
In order for your application to use the same instance of the Auth0 SDK you will need to provide a wrapper that initializes the library. One is provided for you in the `auth0-react-spa.js` that you just created.

Here is an example adding it to a `src/index.js` file

```jsx
// src/index.js
import React from "react";
import ReactDOM from "react-dom";
import App from "./App";
import { Auth0Provider } from "./auth0-react-spa";
import config from "./auth0Config.json";

ReactDOM.render(
  <Auth0Provider
    domain={config.domain}
    client_id={config.clientId}
    redirect_uri={window.location.origin}
    onRedirectCallback={(state) => {
      window.history.replaceState(
        {},
        document.title,
        state && state.targetUrl
          ? state.targetUrl
          : window.location.pathname
      );
    }}
  >
    <App />
  </Auth0Provider>,
  document.getElementById("root")
);
```

## Add Login and Logout Buttons

You will want to add a way for the user to log in and log out of your application.  The wrapper you created earlier provides you 3 helpers to will help you with this `{ isAuthenticated, loginWithRedirect, logout }`.

Here is an example adding it to a Navbar component

```jsx
// src/components/NavBar.js
import React from "react";
import { useAuth0 } from "../auth0-react-spa";

const NavBar = () => {
  const { isAuthenticated, loginWithRedirect, logout } = useAuth0();

  return (
    <div>
      {!isAuthenticated && <button onClick={() => loginWithRedirect({})}>Log in</button>}
      {isAuthenticated && <button onClick={() => logout()}>Log out</button>}
    </div>
  );
};

export default NavBar;
```

## Display the User's Profile

After the user is logged in you can retrieve the user's profile from the `useAuth0` hook.

Here is an example of adding it to a Profile component.

```jsx
// src/components/Profile.js
import React from "react";
import { useAuth0 } from "../auth0-react-spa";

const Profile = () => {
  const { user } = useAuth0();
  if (!user) { return <div></div>; }

  return (
    <div>
      <img src={user.picture} alt="Profile" />

      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <code>{JSON.stringify(user, null, 2)}</code>
    </div>
  );
};

export default Profile;
```

### Use the new components in your App
Now that you have components for your user to login and view their profile information, you will need to manage the application state while it is loading the user.  The `useAuth0` hook provides you with this loading state.

Here is an example of using it in the main application component.

```jsx
// src/App.js
import React from 'react';
import { useAuth0 } from "./auth0-react-spa";
import Navbar from './components/Navbar';
import Profile from './components/Profile';

function App() {
  const { loading } = useAuth0();
  if (loading) { return <div>Loading...</div>; }

  return (
    <div>
      <Navbar />
      <Profile />
    </div>
  );
}

export default App;
```

## What's next
Now your user can log in to your application and view their profile information.  Next up you will learn how to protect a route in your React application to only allow authenticated users to access it.
