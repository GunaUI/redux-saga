# Redux Saga - An alternative to react thunk
* Install redux saga
```jsx
npm install --save redux-saga
```
## Creating our first saga
* you create so-called "sagas" which are essentially kind of functions which you run up on certain actions and which handle all your side effect logic and a side effect simply is something like accessing local storage, reaching out to a server, maybe changing the route or executing a timer.
* sagas/auth.js
```jsx
import { put } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";

function* logoutSaga(action){
    yield localStorage.removeItem('token');
    yield localStorage.removeItem('expirationDate');
    yield localStorage.removeItem('userId');
    yield put({
        type: actionTypes.AUTH_LOGOUT
    })
}
```
* Star ***** in the function is actually turning this function into a so-called generator. Generators are next generation javascript features which are functions which can be executed incrementally, so you can kind of call them and they don't run from start to end immediately but you can pause during function
execution, for example to wait for asynchronous code to finish.
* we should prefix, prepend each step we execute with the yield keyword. This simply means that this step should be executed and
then it will wait for it to finishso if it were an asynchronous action, it wouldn't continue before the step is done.

## Hooking the Saga Up (to the Store and Actions)

* I instead want to also register my sagas and make sure that these are kind of something my store is aware of and can use.
```jsx
import createSagaMiddleware from 'redux-saga';

import { logoutSaga } from './store/sagas/auth';

const sagaMiddleware = createSagaMiddleware();


const store = createStore(rootReducer, composeEnhancers(
    applyMiddleware(thunk, sagaMiddleware)
));

sagaMiddleware.run(logoutSaga);
```
* Now "AUTH_LOGOUT" action run in our redux.right at the start when we built the application, when we started, we run our logout saga which despite
this strange generator stuff going on simply does one thing, it removes all these items from local storage and dispatches the auth logout action.
* Obviously we don't want to run this at application start up, instead we do want to run this code in the place where we previously ran our logout action creator,where we used this,

## Moving Logic from the Action Creator to a Saga
* To move the logic to saga add a new action types and in action call the newly created action types.
```jsx
//actionTypes
export const AUTH_INITIATE_LOGOUT = 'AUTH_INITIATE_LOGOUT';
```
```jsx
// Call newly created actionTypes in your logout action
export const logout = () => {
    // localStorage.removeItem('token');
    // localStorage.removeItem('expirationDate');
    // localStorage.removeItem('userId');
    return {
        type: actionTypes.AUTH_INITIATE_LOGOUT
    };
};
```
* Now the goal is to listen to this newly created action creator and execute our logout saga generator whenever we detect this initiate logout call.
* For that create a new index.js file inside sagas folder.
```jsx
import { takeEvery } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";
import {
  logoutSaga,
} from "./auth";

export function* watchAuth() {
  yield takeEvery(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga);
}
```
* takeEvery,that's another useful function we can use and takeEvery will allow us to listen to certain actions and do something when they occur.
* So in index.js let us not directly call logoutSagas let call this watchAuth
```jsx
import createSagaMiddleware from 'redux-saga';

import { watchAuth } from './store/sagas';

const sagaMiddleware = createSagaMiddleware();


const store = createStore(rootReducer, composeEnhancers(
    applyMiddleware(thunk, sagaMiddleware)
));

sagaMiddleware.run(watchAuth);
```
* with the above changes it will call AUTH_INITIATE_LOGOUT that will in turn call AUTH_LOGOUT
* we can refract the logoutSaga one more step instead of hardcoded action types let create a new action and call that action here

```jsx
import { put } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";

import * as actions from "../actions/index";

function* logoutSaga(action){
    yield localStorage.removeItem('token');
    yield localStorage.removeItem('expirationDate');
    yield localStorage.removeItem('userId');
    yield put(actions.logoutSucceed());
}
```
* And then add this logoutSucceed action
```jsx
export const logoutSucceed = () => {
    return {
        type: actionTypes.AUTH_LOGOUT
    };
};
```
* Now lets add saga for checktimeout action
```jsx
import { put } from "redux-saga/effects";

export function* checkAuthTimeoutSaga(action) {
    yield delay(action.expirationTime * 1000);
    yield put(actions.logout());
}
```
* Now the checkAuthTimeoutSaga created we have to start when we need that for that lets create a new action type and the call that actionType in the auth.js action
```jsx
export const checkAuthTimeout = (expirationTime) => {
    return {
        type: actionTypes.CHECK_TIMEOUT,
        expirationTime : expirationTime
    };
};
```
* Now we have to add this to our watcher...
```jsx
import { takeEvery } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";
import {
    logoutSaga,
    checkAuthTimeoutSaga
} from "./auth";

export function* watchAuth() {
    yield takeEvery(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga);
    yield takeEvery(actionTypes.CHECK_TIMEOUT, checkAuthTimeoutSaga);
}
```
##  Handling Authentication with a Saga

* First step in our auth action is we need to call authStart we have to do the samething in our saga also for that make sure we exporting authStart in actions index file so that all can access that. and other actions also ... as per your need.
```jsx
export {
    auth,
    logout,
    setAuthRedirectPath,
    authCheckState,
    authStart,
    authSuccess,
    checkAuthTimeout,
    authFail
} from './auth';
```
* Next we can start working with our authUser saga 
```jsx
export function* authUserSaga(action) {
    yield put(actions.authStart());
    const authData = {
        email: action.email,
        password: action.password,
        returnSecureToken: true
    };
    let url =
        "https://www.googleapis.com/identitytoolkit/v3/relyingparty/signupNewUser?key=AIzaSyB5cHT6x62tTe-g27vBDIqWcwQWBSj3uiY";
    if (!action.isSignup) {
        url =
        "https://www.googleapis.com/identitytoolkit/v3/relyingparty/verifyPassword?key=AIzaSyB5cHT6x62tTe-g27vBDIqWcwQWBSj3uiY";
    }
    try {
        const response = yield axios.post(url, authData);

        const expirationDate = yield new Date(
        new Date().getTime() + response.data.expiresIn * 1000
        );
        yield localStorage.setItem("token", response.data.idToken);
        yield localStorage.setItem("expirationDate", expirationDate);
        yield localStorage.setItem("userId", response.data.localId);
        yield put(
        actions.authSuccess(response.data.idToken, response.data.localId)
        );
        yield put(actions.checkAuthTimeout(response.data.expiresIn));
    } catch (error) {
        yield put(actions.authFail(error.response.data.error));
    }
}
```
* Now we can use the saga way of auth , so that we can get rid of old action logics. just create a new action types and then call in auth.js actions as follows
```jsx
export const auth = (email, password, isSignup) => {
    // return dispatch => {
    //     dispatch(authStart());
    //     const authData = {
    //         email: email,
    //         password: password,
    //         returnSecureToken: true
    //     };
    //     let url = 'https://www.googleapis.com/identitytoolkit/v3/relyingparty/signupNewUser?key=AIzaSyB5cHT6x62tTe-g27vBDIqWcwQWBSj3uiY';
    //     if (!isSignup) {
    //         url = 'https://www.googleapis.com/identitytoolkit/v3/relyingparty/verifyPassword?key=AIzaSyB5cHT6x62tTe-g27vBDIqWcwQWBSj3uiY';
    //     }
    //     axios.post(url, authData)
    //         .then(response => {
    //             const expirationDate = new Date(new Date().getTime() + response.data.expiresIn * 1000);
    //             localStorage.setItem('token', response.data.idToken);
    //             localStorage.setItem('expirationDate', expirationDate);
    //             localStorage.setItem('userId', response.data.localId);
    //             dispatch(authSuccess(response.data.idToken, response.data.localId));
    //             dispatch(checkAuthTimeout(response.data.expiresIn));
    //         })
    //         .catch(err => {
    //             dispatch(authFail(err.response.data.error));
    //         });
    // };
    return {
        type: actionTypes.AUTH_USER,
        email: email,
        password: password,
        isSignup: isSignup
    };
};
```
* Make sure you hookup the newly created saga with watcher
```jsx
import { takeEvery } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";
import {
    logoutSaga,
    authUserSaga,
    checkAuthTimeoutSaga
} from "./auth";

export function* watchAuth() {
    yield takeEvery(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga);
    yield takeEvery(actionTypes.AUTH_USER, authUserSaga);
    yield takeEvery(actionTypes.CHECK_TIMEOUT, checkAuthTimeoutSaga);
}
```
## Handling Auto-Sign-In with a Saga

```jsx
export function* authCheckStateSaga(action){

        const token = yield localStorage.getItem('token');
        if (!token) {
            yield put(actions.logout());
        } else {
            const expirationDate = yield new Date(localStorage.getItem('expirationDate'));
            if (expirationDate <= new Date()) {
                yield put(actions.logout());
            } else {
                const userId = yield localStorage.getItem('userId');
                yield put(actions.authSuccess(token, userId));
                yield put(actions.checkAuthTimeout((expirationDate.getTime() - new Date().getTime()) / 1000 ));
            }   
        }

}
```
* We have to make this hook up and we want to make sure to run this when we start this app.
```jsx
export const authCheckState = () => {
    // return dispatch => {
    //     const token = localStorage.getItem('token');
    //     if (!token) {
    //         dispatch(logout());
    //     } else {
    //         const expirationDate = new Date(localStorage.getItem('expirationDate'));
    //         if (expirationDate <= new Date()) {
    //             dispatch(logout());
    //         } else {
    //             const userId = localStorage.getItem('userId');
    //             dispatch(authSuccess(token, userId));
    //             dispatch(checkAuthTimeout((expirationDate.getTime() - new Date().getTime()) / 1000 ));
    //         }   
    //     }
    // };
    return {
        type: actionTypes.AUTH_CHECK_STATE,
    };
};
```
* We need to add this to watch lister
```jsx
import { takeEvery } from "redux-saga/effects";

import * as actionTypes from "../actions/actionTypes";
import {
    logoutSaga,
    authUserSaga,
    checkAuthTimeoutSaga,
    authCheckStateSaga,
} from "./auth";

export function* watchAuth() {
    yield takeEvery(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga);
    yield takeEvery(actionTypes.AUTH_USER, authUserSaga);
    yield takeEvery(actionTypes.CHECK_TIMEOUT, checkAuthTimeoutSaga);
    yield takeEvery(actionTypes.AUTH_CHECK_STATE, authCheckStateSaga);
}
```

* Apply the same saga logic to order and burgerBuilder actions.

## Why saga useful ??
* One advantage is our action creators will looks very clean and lean
* Here we can handle our API call , localstorage, this will leads to leaner action creators.

## Diving Deeper into Sagas
* you can import it from redux saga effects and it's called call. Call is a function which allows you to call some function on some object, so you could rewrite this line here where we remove the item by executing yield call and then as a first argument, pass an array where the first element is local storage
but now don't call remove item but instead as a second element, pass that function you want to execute on it. So remove item, written exactly as it is down here,
```jsx
import { put, call } from "redux-saga/effects";

function* logoutSaga(action){
    yield call([localStorage,'removeItem'], 'token');
    // Above code can be more testable than the below one... Apply the same to others also
    yield localStorage.removeItem('token');
}
```
*  If we want to test generators we should follow call method..like the same logic there is one more useful function all
```jsx

import { takeEvery, all } from "redux-saga/effects";

export function* watchAuth() {
    yield all([
         takeEvery(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga),
         takeEvery(actionTypes.AUTH_USER, authUserSaga),
         takeEvery(actionTypes.CHECK_TIMEOUT, checkAuthTimeoutSaga),
         takeEvery(actionTypes.AUTH_CHECK_STATE, authCheckStateSaga),
    ])
    
}

```
* So all is another nice option if you want to run multiple generators or multiple tasks, don't have to be generators, multiple tasks simultaneously.
Again you could use it for the other watchers too

* we can also use takeLatest and if we use that here on purchase burger, takeLatest will automatically cancel any ongoing executions of purchaseBurgerSaga and always only execute the latest one.

```jsx
import { takeEvery, all, takeLatest } from "redux-saga/effects";

 yield takeLatest(actionTypes.AUTH_INITIATE_LOGOUT, logoutSaga);
```
* So that's another nice addition which you might need from time to time to make sure that only one of these processes here is going on at a time
* Refer : https://redux-saga.js.org/docs/introduction/BeginnerTutorial.html
