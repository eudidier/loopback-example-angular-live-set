# Angular Live Set: Example

## What is LiveSet?

[LiveSet](https://github.com/strongloop/angular-live-set) is an angular module that allows you to display an always up to date collection of objects.

![gif](https://cloud.githubusercontent.com/assets/462228/8607451/bf46c74c-2647-11e5-9a72-f458a304ecd1.gif)

## Lets see the code!

**Favorite Colors**

This is a snippet from the [colors example](https://github.com/strongloop/angular-live-set-example/blob/feature/init/client/modules/color/color-list.controller.js) that demonstrates a basic
`LiveSet`.

```js
var src = new EventSource('/api/colors/change-stream');
var changes = createChangeStream(src);
var set;

Color.find().$promise.then(function(results) {
  set = new LiveSet(results, changes);
  $scope.colors = set.toLiveArray();
});
```

**Live Drawing**

The drawing example creates a `LiveSet` in a similar way. The rest of the controller is fairly simple and similar to the snippet above.

The draw method uses a service provided by the [loopback-angular-sdk](http://docs.strongloop.com/display/LB/AngularJS+JavaScript+SDK) to create additional points in the drawing. This data is streamed to other browser clients.

```js
$scope.draw = function(e) {
  if($scope.drawing) {
    Circle.create({
      x: e.offsetX,
      y: e.offsetY,
      r: Math.floor(10 * Math.random())
    });
  }
}
```

**Streaming Chart**

The streaming chart does not use the `LiveSet` class at all. It demonstrates how to stream data from the server to a client.

```js
var src = new EventSource('/api/process/memory');
var changes = createChangeStream(src);

changes.on('data', function(update) {
  // add the new data to the chart
});
```

See the entire chart example code [here](https://github.com/strongloop/angular-live-set-example/tree/feature/init/client/modules/chart).

## Creating the Colors Example

**Step 1: Dependencies**

You will need the following node modules:

 - loopback
 - your favorite gulp or grunt plugins
 - gulp-loopback-angular (or the grunt version)

Also the following bower packages:

 - angular
 - angular-resource
 - angular-live-set

This tutorial assumes you are aware of setting up a basic angular application and
its dependencies. You can use gulp or grunt or nothing at all.

Once you have your angular application running in a browser including the dependencies
mentioned above, you can start using the `lbServices` and `LiveSet` modules to
build the application.

**Step 2: The API**

This example was created first by running `slc loopback`. If you are unfamiliar
with creating LoopBack apps read more about it here.

Once you have the LoopBack API scaffolded, you can add a model. We'll start with
a simple `Color` model just to get some data on the page.

```
slc loopback:model
```

Create the property `val`. Select `string` for the type. Add another property
named `votes`. This should be a `number`.

Generate your `lb-services.js` angular module using the **loopback-angular-sdk**.
This will give you access to the `Color` resource in your angular app. Make sure
you include the script tag and register the module as an angular dependency for
the `lbServices` module (generated by loopback-angular-sdk). You also must include
the source for the `ngResource` module (which you should have installed as part
of step 1).

**Step 3: The Controller**

Now that you have a `Color` model API and the **angular-live-set** module
available from your angular app, you can create a simple controller for interacting
with the `Color` data.

Start with a simple template that renders an array of color objects. I'll assume
you know how to do this. The template should look a bit like this:

```html
<div ng-controller="ColorCtrl">
  <div ng-repeat="color in colors">
    <button
      ng-click="upvote(color.id)"
      style="background: {{ color.val }}">{{ color.votes }}</button>
  </div>
</div>
```

Within our controller we need to create a `LiveSet` of colors as well as
implement the `createColor()` and `upvote()` methods. It should look something
like this.

```js
function ColorCtrl($scope, createChangeStream, LiveSet, Color) {
  $scope.upvote = function(id) {
    Color.upvote({id: id});
  }

  $scope.newColor = 'red';

  $scope.createColor = function() {
    Color.create({val: $scope.newColor, votes: 0});
  }
}
```

And in our model source code we need to add the upvote method:

```js
Color.upvote = function(id, cb) {
  Color.findById(id, function(err, color) {
    if(err) return cb(err);
    color.votes += 1;
    color.save(cb);
  });
};

Color.remoteMethod('upvote', {
  isStatic: true,
  accepts: {arg: 'id', type: 'number'}
});
```

**Step 4: The LiveSet**

The last step is to add the actual live set.

```js
// this should be added to our controller
var changeStreamUrl = '/api/colors/subscription?_format=event-source';
var src = new EventSource(changeStreamUrl);
var changes = createChangeStream(src);
var set;

Color.find().$promise.then(function(colors) {
  set = new LiveSet(colors, changes);
  $scope.colors = set.toLiveArray();
});
```

The code above creates a `LiveSet` from a `ChangeStream`. The `LiveSet` is a
read only collection of data. The items in the set will be updated as changes
are written to the `ChangeStream`.

Since the change stream was created with an `EventSource` the changes will be
written from the server **as they are made**. This will keep the data in the
`LiveSet` up to date.

Also, the `LiveSet` will make sure that changes are applied to the `$scope`.
This means for many use cases, you can create a `LiveSet` as a view of the data
and use the model api (eg. `Color`) to modify the data. Once the change has been
made on the server, the change will be made to your `LiveSet`.

**Compatibility**

`EventSource` is not available in all browsers. However, there are [several
polyfills available](http://bower.io/search/?q=eventsource).

## Run the Examples

Install the dependencies:

```
npm install && bower install
# make sure you have gulp installed globally... or
npm install gulp -g
```

Event streams don't work with Node compression. To disable compression, delete the entry from `server/middleware.json` so it looks like this:
```
...
"compression": {
  "enabled":false
},
...
```

Build and run the example:

```
gulp serve
```

Open two separate browser windows and navigate to `http://localhost:3000` in both.

---

[More LoopBack examples](https://github.com/strongloop/loopback-example)
