#fg-practices
4Geeks' guide to clean, structured code using AngularJS

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
    scope: {
      alias: '='
    }
    controller: function() {
      this.exports = {};

      this.exports.move = function () {/*...*/};
    },
    link: function (scope, element, attrs, controller) {
      scope.alias = controller.exports;
    }
  }
}
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
      return UserResource.get({ id: 1});
    }
  }
})

/* your controller injects the resolved resource as a dependency */

.controller('ViewCtrl', function($scope, user) {
  $scope.user = user;
});
```
This way the controller isnt present until the `user` resource is loaded, so you have your model when the controller initializes.
