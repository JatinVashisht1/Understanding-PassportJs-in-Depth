# Understanding passport js in detail
- in this article I have generalized the idea of using passport and passport strategies
- We will se what is the work of core passport library and passport strategies
- By reading this Readme one should be able to understand passport to a greater extent
- This article actually sumarized a very good medium article on passport js
    - click [here](https://medium.com/@prashantramnyc/node-js-with-passport-authentication-simplified-76ca65ee91e5) to read the article

## Introduction
- Passport JS framework consite of 2 seperate libraries
    1. Primary PassportJS library (to maintain authenticated users)
    2. Strategy library (to authenticate user)

- Passport JS framework abstracts the login process into 2 seperate parts:
    1. Session Management (done by "Passport JS library")
    2. Authentication (done by "Strategy" library)

## Installing corresponding libraries

### Passport JS
- Session management is done by `PassportJS`
- It does not play any role in authenticating new user
- It works with already authenticated user
- To get started with `PassportJS` run following commands in terminal
```
npm i passport
npm i express-session
```
- the "PassportJS" library depends on `express-session` library, so to use `PassportJS` we have to install `express-session`

### Authentication Library
- It is the secondary library used with core PassportJS library
- User can be authenticated in the following ways
    - Username and password stored in database
    - Against Google, Facebook, Github, etc. account
- For example
```
// to use username and password combo
npm i passport-local

// to authenticate against google account
npm i passport-google-oauth

// to authenticate against facebook account
npm i passport-facebook
```

## Integrating Passport to the express app
### 1. Importing libraries
```
// Import Express
const express = require('express)
const app = express()

// Import main Passport and Express_Session library
const passport = require('passport')
const session = require('express-session)

// Import the secondary "Strategy" library
// we are using passport-local strategy in this case
const LocalStrategy = require('passport-local').Strategy
```

### 2. Initializing Middleware
```
app.use(session({
    secret: "secret",
    resave: false,
    saveUninitialized: true,
}))
// This is the basic express session initialization.

app.use(passport.initialize())
// init passport on every route call.

app.use(passport.session())
// allow passport to use "express-session".
```

### 3. Use Passport to define the Authentication Strategy
```
passport.use(new LocalStrategy(authUser))
// the "authUser" is a function that we will define later 
// it will contain the steps to authenticate a user
// and will return the "authenticated user"
```

### 4. Define "authUser" function to get authenticated Users
- "authUser" callback is required within the LocalStragegy
- It contain three arguments:
    1. user: username of user req.body.user
    2. password: password of user defined in req.body.password
    3. done: function used to pass "authenticated user" to serializeUser() function
#### Understanding `done` function
- the done function pass "authenticatedUser" to serializeUser() function
- Signature of done function is written below
```
done(<error>, <user>)
```
- There can be three states of done function
```
// no error is found but user with same username and password is not present
done(null, false)

// no error found and user is present
done(null, {authenticated_user})

// error occured (proper error handling should be done in this case)
// it is adviced to make an error handler middleware
done("something", false)
```
### 5. Serialize and De-Serialize (authenticated) users
#### SerializedUser
- syntax for serializeUser is written below
```
passport.serializeUser((userObj, done)=>{
    done(null, userObj)
})
```
##### Understanding flow of `SerializeUser`
- **"express-session"** created a **"req.session"** object, when it is invoked via app.use(session{...})
- **"passport"** then adds an additional object "req.session.**passport**" to this "req.session"
- **All the serializeUser() function does is,** it receives the "authenticated use" object from "Strategy",
framework and attach the authenticated user to **"req.session.passport.user.{...}"**
- So in effect during "serializeUser", the PassportJS library adds the authenticated user to end of the **"req.session.passport"** object.
- This is what is meant by serialization
- This allows authenticated user to be "attached" to a unique session
- PassportJS directly maintains authenticated users for each session within the **"req.session.passport.user.{...}"**

#### De-serializeUser
- To get user user details for session we can get it from "req.session.passport.**user.{...}**
- definition of `passport.deserializedUser()`
```
passport.deserializeUser((userObj, done)=>{
    done(null, userObj)
})
```

#### Understanding flow of De-Serializeduser
- PassportJS populates "userObj" in deserializeuser() with the object attached at the end of **"req.session.passport.user.{...}"**
- When the done(null, user) function is called in the deserializeUser(), passport js takes this last object attached to **"req.session.passport.user.{...}"**, and attache it to **"req.user"** i.e **"req.user.{...}"**

- So **"req.user"** will contain the **authenticated user object for that session**, and you can use it in any of the routes in the app.
```
// example of above point
app.get("/dashboard", (req, res)=>{
    res.render("dashboard.ejs", {name: req.user.{...}})
})
```

### 6. Use passport.authenticate() as **middleware on your login route**
- You can use passport.authenticate() function within the app.post() and specify the successRedirect and faliureRedirect
```
app.post("/login", passport.authenticate('local', {
    successRedirect: "/dashboard",
    failureRedirect: "/login",
}))
```
- The 'local' signifies that we are using 'local' strategy.
    - If you are using 'google' or 'facebook' instead of 'local'

### 7. Use "req.isAuthenticated()" function to protect **logged in routes**
- Passport JS conveniently provides a "req.isAuthenticated()" function that
    - returns "true" in case an authenticated user is present in **"req.session.passport.user"** and false otherwise

- `req.isAuthenticated()` is used to protet routes that can be accessed only after a user is logged in eg. dashboard
```
checkAuthenticated = (req, res, next)=>{
    if(req.isAuthenticated()) {return next()}
    res.redirect("login")
}

// now checkAuthenticated function can act as a middleware to protect routes as follows,
app.get("/dashboard", checkAuthenticated, (req, res)=>{
    res.render("dashboard.ejs", {name: req.user.name})
})
```
- Similarly if user has already logged in and attempt to access the "register" or "login" screen, you can direct them to the (protected) "dashboard" screen
```
checkLoggedIn = (req, res, next)=>{
    if(req.isAuthenticated()){
        return res.redirect("/dashboard")
    }
    next()
}
// checkLoggedIn function can be used as middleware and can be used as follows,
app.get("/login", **checkLoggedIn**, (req, res)=>{
    res.render("login.ejs")
})
```

### 8. Use `req.logOut()` to clear the sessions object
- Passport JS provides a `req.logOut` function which when called clears the `req.session.passport` object and removes any attached params
```
app.delete("/logout), (req, res)=>{
    req.logOut()
    res.redirect("/login")
    console.log(`------> User Logged Out`)
}
```
