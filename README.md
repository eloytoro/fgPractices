#fgPractices
4Geeks' guide to clean, structured code using AngularJS

##Module Definitions
```javascript
// avoid
var app = angular.module('app', []);
app.controller();

// recommended
angular
    .module('app', []) // only inject dependencies when initializing
    .config()
```

##Controller definition
Sometimes when defining services, directives or routes you ought to fill the controller field with it's declaration...
```javascript
.route({
    url: '/user/:id',
    controller: function($scope) {/*...*/}
});
```
However this pollutes the file with _lots_ of code and in some circumnstanses it's better if you define that controller somewhere else
```javascript
// app.js
.route({
    url: '/user/:id',
    controller: 'UserCtrl'
});

// controllers/user.js
.controller('UserCtrl', function ($scope) {
    /*...*/
});
```

##Application logic defined within the DOM
Often the average AngularJS developer has to make a tough choice: define application logic within the view or within the view's controller.
Keep in mind that when working with a team usually there are areas of expertise; front-end developers stick to .js files and logic whereas web designers work around .css and .html files.
Things could get very confusing for both parties when someone tries to meet both worlds and it's in your best interest to keep them separate.
It's a futile effort to narrow this to a single example, but you'll get the idea
```html
<div class="container" ng-show="isVisible || isAvailable || checkForModels($index)" />
```
Could be something like
```html
<div class="container" ng-show="shown()" />
```
```javascript
$scope.shown = function () {/* logic goes here */};
```
Same goes for controllers, **avoid** referencing controllers in your DOM.
Usually when this happens is becuase the developer is taking a step away from component-oriented design.
For example, you are creating an administrative panel for some user in your web application:

###Naive approach
```html
<!-- Avoid this -->
<div class="userPanel" ng-controller="UserPanelController" />
```

###Component-oriented approach
```javascript
// Declare a directive
.directive('userPanel', function () {
    return {
        scope: { user: '=' },
        controller: 'UserPanelController'
    };
});
```
```html
<!-- Inside your DOM -->
<user-panel user="user" />
```


##Native javascript objects
AngularJS supports a whole set of services that override native javascript _system_ calls such as `setTimeout` or `window`
We encourage you user these services over the default ones because they can be logged, debuged or updated.
[Check the documentation for a full list of services](https://docs.angularjs.org/api/ng/service)

##The alias pattern
Exposing controller functionality can be a tricky task if you're trying to stablish communication between directives.
Among the techniques that achieve this there are:
###Broadcasting Events
```javascript
// Parent scope
$scope.$broadcast('move', 300);

// Directive logic
$scope.$on('move', function(val) {

});
```
There are obvious downsides to this
- You woult be able to control which directives act upon the event
- Pollution to the event emitter
- Does not expose a clear API to users

###Using a service
```javascript
.service('MyDirectiveAPI', function() {
    this.value = 0;
});

// Directive logic
.directive('MyDirective', ['$scope', 'MyDirectiveAPI', function($scope, MyDirectiveAPI) {
    $scope.$watch('value', function(val) {
        this.value = val;
    });
}]);

// Parent controller
.controller('MyController', ['MyDirectiveAPI', function(MyDirectiveAPI) {
  MyDirectiveAPI.value = 300;
}]);
```
You wouldn't be able to control which directives alter because the value mutates globally to all directives

###The alias pattern (RECOMENDED)
```javascript
.directive('MyDirective', function() {
    return {
        scope: { alias: '@?' }
        controller: function() {
            this.move = function () {/*...*/};
        },
        link: function (scope, element, attrs, controller) {
            if (scope.alias)
                scope.$parent[scope.alias] = controller;
        }
    };
});
```
The only downside about doing this is that the functionality wont be exported until the directive links.

##Views dependant on resource queries
Whenever you're building a view based on a model you're likely to encouter a very simple, very common isue: your model hasnt arrived  to your front-end applicatin at the moment of `onload`
This cause a lot of confussion in your view controller.
Do note that this only applies when the view is absolutely dependant on the model value due for retrieval.

###In-controller ajax query
```javascript
.controller('ViewCtrl', function($scope, UserResource) {
    $scope.user = UserResource.get({ id: 1 });
});
```
The obvious downside to this is that your `$scope.user` promise wont hold the model until it resolves, which means you will have to check constantly its status.

###Route resolving (RECOMMENDED)
Most routers support this feature.
```javascript
.route({
    url: '/user/:id',
    controller: 'ViewCtrl',
    resolve: {
        user: function(UserResource) {
            return UserResource.get({ id: 1}).$promise;
        }
    }
})

/* your controller injects the resolved resource as a dependency */

.controller('ViewCtrl', function($scope, user) {
    $scope.user = user;
});
```
This way the controller isnt present until the `user` resource is loaded, so you have your model when the controller initializes.

##Promise Chaining
If you don't know what a promise is then checkout [domenic's article on promises](https://gist.github.com/domenic/3889970) and [angular's documentation for $q](https://docs.angularjs.org/api/ng/service/$q)
All of angular's asynchronous calls (such as resource queries or http requests) work with promises and we encourage you to squeeze the most out of them.

###Pyramid of Doom
```javascript
step1(function (value1) {
    step2(value1, function(value2) {
        step3(value2, function(value3) {
            step4(value3, function(value4) {
                // Do something with value4
            });
        });
    });
});
```
We've all been part in the everlasting ordeal of perpetuating the pyramid of doom across the lengths of javascript code.

###Using .then (RECOMMENDED)
```javascript
Q.fcall(promisedStep1)
    .then(promisedStep2)
    .then(promisedStep3)
    .then(promisedStep4)
    .then(function (value4) {
        // Do something with value4
    })
    .catch(function (error) {
        // Handle any error from all above steps
    })
    .done();
```
Much more organized, promises allow you to chain them dynamically across the thread that defines them and it's always cool to return promises for others to keep chaining them!
```javascript
.controller('ExampleCtrl', function ($q, $http) {
    this.asyncRequest = function () {
        return $http({/*...*/});
    };

    this.asyncRequest()
        .then(function (data) {
            /*...*/
        });
})
```

###Resource queries
Resources work with promises as well, use them!
```javascript
$scope.users = UserResource.query(function () {
    console.log($scope.users) // shows all queried users
});
console.log($scope.users) // shows an unfulfilled promise
```

##Form Validation
Often at times we're validating many forms across our app, many of which are similar.

But this task might seem slow and tedious because our understanding of angular forms tells us that these have to be validated programatically.

Even thought we're provided with a set of pre-defined directives that help us achieve correctness in our every field there are times where some cases require some kind of special validation, such as values that depend on another values.

But the real problem lies when we're validating many forms upon the same logic and we end up re-defining the same html or javascript code for every instance.

###Naive approach
Most developers are prone to fall for defining form validation logic inside the view's controller.
This doesn't follow the DRY principle and therefore isn't exactly scalable.
```html
<form name="myForm">
    <input type="text" name="myField" ng-model="myValue">
</form>
```
```javascript
// witihin your view's controller
$scope.$watch('myValue', function (val) {
    $scope.myForm.myField.$setValidity(val === 'awesome');
});
```

###Model-defined form validation (RECOMMENDED)
Understanding forms as models comes out natural, most of the times you're creating a form that exposes model values, but letting the form define the model's logic is wrong, it should be the other way around: **models should define how forms are validated**.
Take this directive for example
```javascript
.directive('userForm', function () {
    return {
        restrict: 'A',
        require: ['form?', 'ngForm?'],
        link: function (scope, element, attrs, formCtrl) {
            if (!formCtrl) return;
            formCtrl.myField.$validators.isAwesomeEnough = function (val) {
                return val === 'awesome';
            };
        }
    };
});
```
```html
<form name="myForm" user-form>
    <input type="text" name="myField" ng-model="myValue">
</form>
```
