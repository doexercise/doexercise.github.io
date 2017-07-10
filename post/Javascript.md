<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Javascript, Node.js

<br />

# jQuery 예제 1
## 서버측(Node.js)
```Javascript
var http 	= require('http'),
    express 	= require('express'),
    bodyParser 	= require('body-parser');

var app 	= express();

app.use(bodyParser.urlencoded({
	extended: true
}));

app.engine('html', require('ejs').renderFile);
app.use(express.static(__dirname + '/js'));

app.get('/', function(req, res){
        res.render(__dirname + '/index.html');
});

app.get('/data', function(req, res) {
        var value = { };
        var dataStr = { };

        value.x1 = Math.ceil(Math.random() * 100);
        value.x2 = Math.random() * 100;
        dataStr = JSON.stringify(value);

        res.writeHead(200, { 
                'Content-Type': 'text/json', 
                'Content-Length': Buffer.byteLength(dataStr) });
        res.write(dataStr);
        res.end();
});

var server = http.createServer(app).listen(3000, function(){
	console.log("Http server listening on port " + 3000);
});
```

### 클라이언트 측(jQuery)
```html
<!DOCTYPE html>
<html>
  <head>
    <!-- 이 파일은 /js/smoothie.js 에서 정적으로 제공되는 것이다. -->
    <script type="text/javascript" src="smoothie.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
    <script type="text/javascript">

      // Randomly add a data point every 500ms
      var random1 = new TimeSeries();
      var random2 = new TimeSeries();
      setInterval(function() {
        getData();
      }, 500);
      
      function createTimeline() {
        var chart = new SmoothieChart();
        chart.addTimeSeries(random1, { strokeStyle: 'rgba(255, 0, 0, 1)', fillStyle: 'rgba(255, 0, 0, 0.2)', lineWidth: 2 });
        chart.addTimeSeries(random2, { strokeStyle: 'rgba(0, 255, 255, 1)', fillStyle: 'rgba(0, 255, 255, 0.2)', lineWidth: 2 });
        chart.streamTo(document.getElementById("chart1"), 500);
      }

      // jQuery는 겨우 요거 ㅋㅋ
      function getData() {
        $.get("/data", function (data) {
          console.log(data);
          random1.append(new Date().getTime(), data.x1);
          random2.append(new Date().getTime(), data.x2);
        });
      };

    </script>
  </head>
  <body onload="createTimeline()">

    <canvas id="chart1" width="800" height="200"></canvas>

  </body>
</html>
```
<br />

## [**Table of Contents**](../README.md)