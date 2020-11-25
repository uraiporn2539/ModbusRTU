# modbus
## Basic
### Sensor : XY-MD02
![md](https://user-images.githubusercontent.com/42264167/99418096-3ab2de00-292d-11eb-8023-04067344b3d5.PNG)

จากภาพข้างต้นจะเห็นได้ว่าการอ่านค่าของ Temperature และ Humidity ที่อยู่ในตำแหน่ง 0x0001 และ 0x0002 นั้นถูกเก็บอยู่ใน Input register ซึ่งสามารถใช้คำสั่ง Command Register 0x04 ในการอ่านค่าจาก Input register ได้ 

### modbus-serial (npm)
![](https://user-images.githubusercontent.com/42264167/99420533-e4936a00-292f-11eb-954a-1393efa89a32.PNG)

จาก method ข้างต้นนั้นสามารถใช้ในการอ่านค่า อันมี Function Code = 4 (FC=4) อันหมายถึง Command Register 0x04 ที่ใช้ในการอ่านข้อมูลจาก Input register ซึ่ง method ดังกล่าวนั้น จะรับ Parameter 2 ตัว ได้แก่ address และ length 

ซึ่งในการอ่านค่า Temperature และ Humidity นั้น จะเห็นได้ว่าถูกเก็บอยู่ที่ตำแหน่ง 1 และ 2 ตามลำดับ นั่นหมายความว่าในการอ่านค่าทั้งสอง จะต้องระบุ address เป็น 1 โดยให้อ่านมา 2 ตัว (length=2) จึงจะได้ข้อมูลมาทั้ง Temperature และ Humidity 

## อ่านและจัดเก็บข้อมูล
### การเชื่อมต่อ RaspberryPi
เชื่อมต่อ Sensoe XY-MD02 เข้าสู่ Rasberry Pi ผ่าน RS485 Module ดังภาพ 
![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/xy.jpeg)
เครดิตภาพ https://www.joom.com/th/products/5ca498ce28fc7101019f3549

![](https://user-images.githubusercontent.com/42264167/99426443-a8afd300-2936-11eb-8040-bbcddc504143.jpg)

### การตรวจสอบพอร์ตที่ถูกเชื่อมต่ออยู่
สแกนหา module RS485 ที่เชื่อมต่ออยู่กับ RassberryPi
โดยใช้คำสั่ง 
``` ls /dev ``` 
โดยเชื่อมต่อ module และ ไม่ได้เชื่อมต่อ module จะมีผลลัพธ์แตกต่างกันดังนี้
![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/no.PNG)
รูปนี้ ยังไม่ได้เสียบสาย

![](https://raw.githubusercontent.com/gingkasina/ModbusRTU/master/image/have%20-%20Copy.PNG)
รูปนี้ เสียบสายแล้ว จะเห็นถึงความแตกต่างที่ วงกลมสีแดง คือมี ttyUSB0 เพิ่มขึ้นมา 

### ติดตั้ง modbus-serial
ทำการติดตั้ง modbus-serial ผ่านคำสั่ง 
```
npm install modbus-serial --save
```

แล้วทำการเชื่อมต่อไปยัง sensor โดยใช้คำสั่งดังต่อไปนี้
``` js
var ModbusRTU=require('modbus-serial');
var client=new ModbusRTU();
client.connectRTU("/dev/ttyUSB0",{buadRate: 9600}); //RS485 connected to ttyUSB0
```

### การอ่านค่า 
สามารถทำได้โดยใช้คำสั่ง 
``` js
client.readInputRegisters(1,2).then((data)=>{
    console.log(data);
}).catch((e)=>{
    console.log(e.message);
});
```

### การจัดเก็บข้อมูล
#### Schema และ Model 
Schema และ Model ที่ใช้ในการจัดเก็บข้อมูลแสดงได้ดังต่อไปนี้
```js
var mongoose=require('mongoose');
var Schema=mongoose.Schema;

var modbusSchema=new Schema({
    temperature: Number,
    humidity: Number,
    datetime: String
});

mongoose.model('modbus',modbusSchema);
```

#### การบันทึกข้อมูลลงฐานข้อมูล
ในที่นี้ได้ทำการกำหนดให้ทำการจัดเก็บข้อมูลสู่ฐานข้อมูลทุก ๆ 60000m/s หรือ 1 นาที โดยสามารถกระทำได้โดยมีลักษณะคำสั่งดังต่อไปนี้
```js
var modbusModel=require('mongoose')().model('modbus');
var ModbusRTU=require('modbus-serial');
var client=new ModbusRTU();
client.connectRTU("/dev/ttyUSB0",{buadRate: 9600}); //RS485 connected to ttyUSB0

setInterval(()=>{
    client.readInputRegisters(1,2)
    .then((data)=>{
        var allData=data.data;
        var model={
            temperature: allData[0]/10, //devide by 10 followed datasheet
            humidity: allData[1]/10, //devide by 10 followed datasheet
            datetime: Date.now()
        }
        modbusModel.insertMany(model,(err,docs)=>{
            if(err)console.log(err);
        });
    }).catch((e)=>{
        console.log(e.message);
    })
},60000); 
```

## การอ่านและแสดงผลข้อมูล
### การอ่านข้อมูลอุณหภูมิ ความชื้นและเวลา
ผุ้ทดลองได้ทำการสร้าง request handle method ขั้นมาเพื่อทำการส่งข้อมูลที่ถูกจัดเก็บอยู่บนฐานข้อมูลกลับไป โดยส่งกลับไปเป็นทั้งข้อมูลทั้งหมดและข้อมูลเฉลี่ยย้อนหลังเจ็ดวัน โดยมีลักษณะการทำงานดังต่อไปนี้
```js
    var express=require('express');
    var app=express();
    var mongoose=require('mongoose');
    var modbusModel=mongoose().model('modbus');

    app.get('/api/data',(req,res)=>{
        modbusModel.find({},(err,data)=>{
            if(!err){
                let detail=new Array();
                let tmp={};
                data.sort((a,b)=>{
                    return b.datetime-a.datetime;
                });
                data.forEach((item,index)=>{
                    if(!item.temperature||!item.humidity||item.humidity>100||item.temperature>100)return;
                    let dt=new Date(parseInt(item.datetime));
                    detail.push({'dateTime':dt, 'temperature':item.temperature, 'humidity':item.humidity});
                });
                detail.forEach((item,index)=>{
                    if(isNaN(item.dateTime.getUTCDate())||isNaN(item.dateTime.getUTCMonth())||isNaN(item.dateTime.getUTCFullYear())||!item.temperature||!item.humidity)return;
                    let obj=tmp[item.dateTime.getUTCDate()+'/'+(item.dateTime.getUTCMonth()+1)+'/'+item.dateTime.getUTCFullYear()]=tmp[item.dateTime.getUTCDate()+'/'+(item.dateTime.getUTCMonth()+1)+'/'+item.dateTime.getUTCFullYear()]||{count:0, totalTemperature:0, totalHumidity:0};
                    obj.count++;
                    obj.totalTemperature+=item.temperature;
                    obj.totalHumidity+=(item.humidity/5);
                });
                let result=Object.entries(tmp).map(function(entry){
                    return {date: entry[0], temperature: entry[1].totalTemperature/entry[1].count, humidity: entry[1].totalHumidity/entry[1].count};
                });
                res.json({detail:detail, average: result.slice(0,7)});
            }
        });
    });
```

### การแสดงผลข้อมูลเป็นกราฟและตารางข้อมูล
ดำเนินการเหมือน [temp_rtc](https://github.com/thanaponyalan/temp_rtc) ทุกประการแตกต่างกันเพียงไฟล์ tempChart.js อันถูกปรับเปลี่ยนเป็นดังนี้
```js
angular.module("app", ["chart.js","datatables"])
// Optional configuration
.config(['ChartJsProvider', function (ChartJsProvider) {
    // Configure all charts
    ChartJsProvider.setOptions({
        chartColors: ['#FF5252', '#4682B4'],
        responsive: true
    });
    // Configure all line charts
    ChartJsProvider.setOptions('line', {
        showLines: true,
    });
}])
.controller("LineCtrl", ['$scope', '$interval', '$http', function ($scope, $interval, $http) {
    $scope.onClick = function (points, evt) {
        console.log(points, evt);
    };
    $scope.options={
        legend:{
            display: true
        },
        elements:{
            line:{
                fill: false
            }
        }
    }

    var refresh=()=>{
        $http.get('http://raspberrypi.local:3000/api/modbus').then((response)=>{
            let labels=new Array();
            let temperature=new Array();
            let humidity=new Array();
            angular.forEach(response.data.average,function(value, key){
                labels.push(value.date);
                temperature.push(value.temperature.toFixed(2));
                humidity.push(value.humidity.toFixed(2));
            });
            labels.reverse();
            temperature.reverse();
            humidity.reverse();
            $scope.labels=labels;
            $scope.series=['Temperature (*C)','Humidity (1:5%rh)'];
            $scope.data=[temperature,humidity];
        });
    };
    refresh();
    $interval(refresh,60000)
}])
.controller('DatatablesCtrl', ['$scope','$interval','$http', function ($scope, $interval, $http) {
    var refresh=()=>{
        $http.get('http://raspberrypi.local:3000/api/modbus').then((response)=>{
            $scope.details=response.data.detail;
        });
    };
    refresh();
    $interval(refresh,60000);
}])
.filter('formatAsDate',function(){
    return function(dateTime){
        return moment(dateTime).format('YYYY-MM-DD HH:mm');
    }
});
```