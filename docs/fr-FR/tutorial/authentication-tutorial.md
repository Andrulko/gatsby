---
title: Making a site with user authentication
---

Sometimes, you need to create a site with gated content, available only to authenticated users. Using Gatsby, you may achieve this using the concept of [client-only routes](/docs/building-apps-with-gatsby/#client-only-routes), to define which pages a user can view only after logging in.

## Prerequisites

You should have already configured your environment to be able to use the `gatsby-cli`. A good starting point is the [main tutorial](/tutorial).

## Security notice

In production, you should use a tested and robust solution to handle the authentication. [Auth0](https://www.auth0.com), [Firebase](https://firebase.google.com), and [Passport.js](http://passportjs.org) are good examples. This tutorial will only cover the authentication workflow, but you should take the security of your app as seriously as possible.

## Building your Gatsby app

Start by creating a new Gatsby project using the barebones `hello-world` starter:

```shell
gatsby new gatsby-auth gatsbyjs/gatsby-starter-hello-world
cd gatsby-auth
```

Create a new component to hold the links. For now, it will act as a placeholder:

```jsx:title=src/components/nav-bar.js import React from "react" import { Link } from "gatsby"

export default () => ( <div style={{ display: "flex", flex: "1", justifyContent: "space-between", borderBottom: "1px solid #d1c1e0", }} > <span>You are not logged in</span>

    <nav>
      <Link to="/">Home</Link>
      {` `}
      <Link to="/">Profile</Link>
      {` `}
      <Link to="/">Logout</Link>
    </nav>
    

</div> )

    <br />And create the layout component that will wrap all pages and display navigation bar:
    
    ```jsx:title=src/components/layout.js
    import React from "react"
    
    import NavBar from "./nav-bar"
    
    const Layout = ({ children }) =&gt; (
      &lt;&gt;
        &lt;NavBar /&gt;
        {children}
      &lt;/&gt;
    )
    
    export default Layout
    

Lastly, change the index page to use layout component:

```jsx:title=src/pages/index.js import React from "react"

import Layout from "../components/layout" // highlight-line

// highlight-start export default () => ( <Layout> 

# Hello world!</Layout> ) // highlight-end

    <br />## Authentication service
    
    For this tutorial you will use a hardcoded user/password. Create the folder `src/services` and add the following content to the file `auth.js`:
    
    ```javascript:title=src/services/auth.js
    export const isBrowser = () =&gt; typeof window !== "undefined"
    
    export const getUser = () =&gt;
      isBrowser() && window.localStorage.getItem("gatsbyUser")
        ? JSON.parse(window.localStorage.getItem("gatsbyUser"))
        : {}
    
    const setUser = user =&gt;
      window.localStorage.setItem("gatsbyUser", JSON.stringify(user))
    
    export const handleLogin = ({ username, password }) =&gt; {
      if (username === `john` && password === `pass`) {
        return setUser({
          username: `john`,
          name: `Johnny`,
          email: `johnny@example.org`,
        })
      }
    
      return false
    }
    
    export const isLoggedIn = () =&gt; {
      const user = getUser()
    
      return !!user.username
    }
    
    export const logout = callback =&gt; {
      setUser({})
      callback()
    }
    

## Creating client-only routes

At the beginning of this tutorial, you created a "hello world" Gatsby site, which includes the `@reach/router` library. Now, using the [@reach/router](https://reach.tech/router/) library, you can create routes available only to logged-in users. This library is used by Gatsby under the hood, so you don't even have to install it.

First, create `gatsby-node.js` in root directory of your project. You will define that any route that starts with `/app/` is part of your restricted content and the page will be created on demand:

```javascript:title=gatsby-node.js // Implement the Gatsby API “onCreatePage”. This is // called after every page is created. exports.onCreatePage = async ({ page, actions }) => { const { createPage } = actions

// page.matchPath is a special key that's used for matching pages // only on the client. if (page.path.match(/^\/app/)) { page.matchPath = "/app/*"

    // Update the page.
    createPage(page)
    

} }

    <br />&gt; Note: There is a convenient plugin that already does this work for you: [gatsby-plugin-create-client-paths](/packages/gatsby-plugin-create-client-paths)
    
    Now, you must create a generic page that will have the task to generate the restricted content:
    
    ```jsx:title=src/pages/app.js
    import React from "react"
    import { Router } from "@reach/router"
    import Layout from "../components/layout"
    import Profile from "../components/profile"
    import Login from "../components/login"
    
    const App = () =&gt; (
      &lt;Layout&gt;
        &lt;Router&gt;
          &lt;Profile path="/app/profile" /&gt;
          &lt;Login path="/app/login" /&gt;
        &lt;/Router&gt;
      &lt;/Layout&gt;
    )
    
    export default App
    

Next, add the components regarding those new routes. The profile component to show the user data:

```jsx:title=src/components/profile.js import React from "react"

const Profile = () => ( <> 

# Your profile

- Name: Your name will appear here
- E-mail: And here goes the mail</> )

export default Profile

    <br />The login component will handle - as you may have guessed - the login process:
    
    ```jsx:title=src/components/login.js
    import React from "react"
    import { navigate } from "gatsby"
    import { handleLogin, isLoggedIn } from "../services/auth"
    
    class Login extends React.Component {
      state = {
        username: ``,
        password: ``,
      }
    
      handleUpdate = event =&gt; {
        this.setState({
          [event.target.name]: event.target.value,
        })
      }
    
      handleSubmit = event =&gt; {
        event.preventDefault()
        handleLogin(this.state)
      }
    
      render() {
        if (isLoggedIn()) {
          navigate(`/app/profile`)
        }
    
        return (
          &lt;&gt;
            &lt;h1&gt;Log in&lt;/h1&gt;
            &lt;form
              method="post"
              onSubmit={event =&gt; {
                this.handleSubmit(event)
                navigate(`/app/profile`)
              }}
            &gt;
              &lt;label&gt;
                Username
                &lt;input type="text" name="username" onChange={this.handleUpdate} /&gt;
              &lt;/label&gt;
              &lt;label&gt;
                Password
                &lt;input
                  type="password"
                  name="password"
                  onChange={this.handleUpdate}
                /&gt;
              &lt;/label&gt;
              &lt;input type="submit" value="Log In" /&gt;
            &lt;/form&gt;
          &lt;/&gt;
        )
      }
    }
    
    export default Login
    

Though the routing is working now, you still can access all routes without restriction.

## Controlling private routes

To check if a user can access the content, you can wrap the restricted content inside a PrivateRoute component:

```jsx:title=src/components/privateRoute.js import React, { Component } from "react" import { navigate } from "gatsby" import { isLoggedIn } from "../services/auth" class PrivateRoute extends Component { componentDidMount() { const { location } = this.props let noOnLoginPage = location.pathname !== `/app/login` if (!isLoggedIn() && noOnLoginPage) { navigate("/app/login") return null } } render() { const { component: Component, ...rest } = this.props return <Component {...rest} /> } }

export default PrivateRoute

    <br />And now you can edit your Router to use the PrivateRoute component:
    
    ```jsx:title=src/pages/app.js
    import React from "react"
    import { Router } from "@reach/router"
    import Layout from "../components/layout"
    import PrivateRoute from "../components/privateRoute" // highlight-line
    import Profile from "../components/profile"
    import Login from "../components/login"
    
    const App = () =&gt; (
      &lt;Layout&gt;
        &lt;Router&gt;
          {/* highlight-next-line */}
          &lt;PrivateRoute path="/app/profile" component={Profile} /&gt;
          &lt;Login path="/app/login" /&gt;
        &lt;/Router&gt;
      &lt;/Layout&gt;
    )
    
    export default App
    

## Refactoring to use new routes and user data

With the client-only routes in place, you must now refactor some files to account for the user data available.

The navigation bar will show the user name and logout option to registered users:

```jsx:title=src/components/nav-bar.js import React from "react" import { Link, navigate } from "gatsby" // highlight-line import { getUser, isLoggedIn, logout } from "../services/auth" // highlight-line

// highlight-start export default () => { const content = { message: "", login: true } if (isLoggedIn()) { content.message = `Hello, ${getUser().name}` } else { content.message = "You are not logged in" } return ( // highlight-end <div style={{ display: "flex", flex: "1", justifyContent: "space-between", borderBottom: "1px solid #d1c1e0", }} > <span>{content.message}</span> {/* highlight-line */} <nav> 

<Link to="/" />
Home</Link> {
``} 

<Link to="/app/profile" />
Profile</Link> {/* highlight-line 
*/} {``} {/* highlight-start */} {isLoggedIn() ? ( <a href="/" onClick={event => { event.preventDefault() logout(() => navigate(`/app/login`)) }} > Logout </a> ) : null} {/* highlight-end */} </nav> </div> ) } // highlight-line

    <br />The index page will suggest to login or check the profile accordingly:
    
    ```jsx:title=src/pages/index.js
    import React from "react"
    import { Link } from "gatsby" // highlight-line
    import { getUser, isLoggedIn } from "../services/auth" // highlight-line
    
    import Layout from "../components/layout"
    
    export default () =&gt; (
      &lt;Layout&gt;
        {/* highlight-start */}
        &lt;h1&gt;Hello {isLoggedIn() ? getUser().name : "world"}!&lt;/h1&gt;
        &lt;p&gt;
          {isLoggedIn() ? (
            &lt;&gt;
              You are logged in, so check your{" "}
              &lt;Link to="/app/profile"&gt;profile&lt;/Link&gt;
            &lt;/&gt;
          ) : (
            &lt;&gt;
              You should &lt;Link to="/app/login"&gt;log in&lt;/Link&gt; to see restricted
              content
            &lt;/&gt;
          )}
        &lt;/p&gt;
        {/* highlight-end */}
      &lt;/Layout&gt;
    )
    

And the profile will show the user data:

```jsx:title=src/components/profile.js import React from "react" import { getUser } from "../services/auth" // highlight-line

const Profile = () => ( <> 

# Your profile

{/* highlight-start */} 

- Name: {getUser().name}
- E-mail: {getUser().email} {/* highlight-end */} </> )

export default Profile ```

You should now have a complete authentication workflow, functioning with both login and a user-restricted area!

## Further reading

If you want to learn more about using production-ready auth solutions, these links may help:

- [Gatsby repo simple auth example](https://github.com/gatsbyjs/gatsby/tree/master/examples/simple-auth)
- [A Gatsby email *application*](https://github.com/DSchau/gatsby-mail), using React Context API to handle authentication
- [The Gatsby store for swag and other Gatsby goodies](https://github.com/gatsbyjs/store.gatsbyjs.org)
- [Building a blog with Gatsby, React and Webtask.io!](https://auth0.com/blog/building-a-blog-with-gatsby-react-and-webtask/)
- [JAMstack PWA — Let’s Build a Polling App. with Gatsby.js, Firebase, and Styled-components Pt. 2](https://medium.com/@UnicornAgency/jamstack-pwa-lets-build-a-polling-app-with-gatsby-js-firebase-and-styled-components-pt-2-9044534ea6bc)
- [JAMstack Hackathon Starter - Authenticated Gatsby app starter with Netlify Identity](/starters/sw-yx/jamstack-hackathon-starter)
- [Learn With Jason Livestream: How to use Netlify Identity and Netlify Functions (with Shawn Wang)](https://www.youtube.com/watch?v=vrSoLMmQ46k&feature=youtu.be)