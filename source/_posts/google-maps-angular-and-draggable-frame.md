title: 'Google Maps, Angular and Draggable Frame'
tags:
  - 'Draggable div'
  - 'Google maps overlay'
  - 'Update content with drag'
id: 89
categories:
  - Angular.js
date: 2013-09-20 14:55:25
---

Probably not the best title for this post, but one image will explain it:

[![Screen Shot 2013-09-20 at 1.54.56 PM](http://noamshemesh.files.wordpress.com/2013/09/screen-shot-2013-09-20-at-1-54-56-pm.png?w=300)](http://noamshemesh.files.wordpress.com/2013/09/screen-shot-2013-09-20-at-1-54-56-pm.png)Only the markers in the phone are shown, the user can move the phone frame and see others markers.

The main idea is to create a div that would update angular with its current position (and sizes, if not fixed), and then update the markers on the map to be visible if they are inside the frame, otherwise set them to be invisible.

<!--more-->An ideal angular library for google maps is [nlaplante's angular-google-maps](https://github.com/nlaplante/angular-google-maps). The library itself is very good, although it doesn't export the google's main map object, what I believe he did for a reason, encapsulation for that matter, and this is probably the correct way of doing it, but wasn't enough for me. I understood that wrapping all the objects and the events that I need probably will get me nothing. So I [forked](https://github.com/noamshemesh/angular-google-maps) it, and also added padding to the "fit" functionality, and export the google's markers objects too.

For the draggable functionality I use [jquery-ui,](http://jqueryui.com/draggable/) which I wasn't familiar with, but it is easy to use and do what you want it to do.

Let's start with some code, the main page that the user sees is index.jade, that using the [jade](http://jade-lang.com/) language, that is seamlessly compiled to html

index.jade:

``` javascript
#draggable(draggable, xpos='xpos', ypos='ypos')
  img(id='phone')
google-map.map(center=&quot;center&quot;, fit=&quot;true&quot;, draggable=&quot;true&quot;, zoom=&quot;zoom&quot;, markers=&quot;markers&quot;, mark-click=&quot;false&quot;, map=&quot;map&quot;)
```

The first line of the index.jade is the definition of the phone frame that is draggable, you can see that there are 2 "draggable" elements, the first is the id of the object, that is needed for the jquery-ui in order to define it as draggable and for this css:

app.scss:

``` css
  #draggable {
    height: 516px;
    width: 300px;
    z-index: 2;
    position: absolute;
    top: 0;
    left: 0;

    #phone {
      height: 516px;
      width: 300px;
      background-image: url(&quot;/img/android_frame.png&quot;);
      background-size: 300px 516px;
      background-repeat: no-repeat;
    }
  }
```

Notice the z-index of the div.

Back to the draggable definition, the second draggable word, with the other attributes, are the interesting ones, they related directly to this angular directive:

controller.js:

``` javascript
app.directive('draggable', function () {
  return {
    restrict: 'A',
    link: function (scope, element, attrs) {
      var update = function (event, ui) {
        scope.$eval(attrs.xpos + '=' + ui.position.left);
        scope.$eval(attrs.ypos + '=' + ui.position.top);
        scope.$apply();
      };
      var lastUpdate = 0;
      var updateEveryXMillis = function (event, ui) {
        if (new Date().getTime() - lastUpdate &gt;= 100) {
          update(event, ui);
          lastUpdate = new Date().getTime();
        }
      }

      $('#draggable').draggable({
        cursor: &quot;drag&quot;,
        stop: update,
        drag: updateEveryXMillis
      });
    }
  };
});
```

The directive itself initialize the jquery-ui draggable functionality on the element, the element parameter of link function didn't has the draggable method, causing `TypeError: Object [object Object] has no method 'draggable'`. When calling to draggable method, I define what to do when the user drags and when he stops, the logic of these functions is simple, update xpos and ypos of the $scope (they are under the scope cause I bound them in the jade above), and now, all we have to do is to update the markers visibility:

controller.js:

``` javascript
var app = angular.module(&quot;map&quot;, [&quot;google-maps&quot;]);
app.controller('MapCtrl', ['$scope', '$http', function ($scope, $http) {
  angular.extend($scope, {
    center: {
      latitude: 0,
      longitude: 0
    },
    markers: [],
    zoom: 8
  });

  function getLocations() {
    $http.get('/locations').success(function (data) {
      var markers = []
      _.forEach(data.locations, function (location) {
        markers.push({ latitude: location.latitude, longitude: location.longitude, visible: false });
      });
      $scope.markers = markers;
    });
  }

  getLocations();
  setInterval(getLocations, 5000);

  var phone = $('#phone')
  var phoneWidth = phone.width();
  var phoneHeight = phone.height();

  function updateMarkerVisibility(marker, xpos, ypos) {
    var markerPoint = $scope.map.overlay.getProjection().
        fromLatLngToContainerPixel(marker.getPosition());
    marker.setVisible(markerPoint.x &gt;= xpos &amp;&amp; markerPoint.x &lt;= xpos + phoneWidth &amp;&amp;
        markerPoint.y &gt;= ypos &amp;&amp; markerPoint.y &lt;= ypos + phoneHeight);
  }

  $scope.$watch('map._markers', function (newMarkers) {
    _.forEach(newMarkers, function (marker) {
      updateMarkerVisibility(marker, $scope.xpos, $scope.ypos);
    })
  })

  $scope.$watch('xpos', function (newValue, oldValue) {
    if ($scope.map &amp;&amp; $scope.map._markers) {
      _.forEach($scope.map._markers, function (marker) {
        updateMarkerVisibility(marker, newValue, $scope.ypos);
      })
    }
  })

  $scope.$watch('ypos', function (newValue, oldValue) {
    if ($scope.map &amp;&amp; $scope.map._markers) {
      _.forEach($scope.map._markers, function (marker) {
        updateMarkerVisibility(marker, $scope.xpos, newValue);
      })
    }
  })
}]);

```

Here I watch the xpos, ypos of the scope and the markers of google's map that have been updated when markers got updated in the interval function by the angular-google-maps library.
When any delegate of the watch function is called it decides if each marker need to be shown.
If you want to support zoom you need to watch its changes too.

The angular extend, is for the angular-google-maps library that mentioned above.

Don't forget to tell angular what is your controller in the jade file.

Noam.