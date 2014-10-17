#fg-practices
4Geeks' guide to clean, structured code using AngularJS

##The alias pattern
Exposing controller functionality can be a tricky task if you're trying to stablish communication between directives.
Among the techniques that achieve this are:
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
.controller('MyController, ['MyDirectiveAPI', function(MyDirectiveAPI) {
  MyDirectiveAPI.value = 300;
}]);
```
This won't be able to control which directives change whenever the value mutates as it's global to all directives

###(RECOMENDED) The alias pattern
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
The only downside about doing this is that exported content wont be able to be (safely) accessed until the directive links
It's recommended that you only export functionality that would be used after the linking process happens
