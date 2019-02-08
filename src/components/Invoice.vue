<template>
  <v-container>
    <v-layout>
      <v-flex xs12>
        <v-btn @click="startScan">Scan Invoice</v-btn>
        <v-btn @click="initialVarFill">Fill Variable</v-btn>
      </v-flex>
    </v-layout>
    <v-layout text-xs-center wrap>
      <v-flex xs8>
        <v-stage ref="stage" :config="configKonva">
        </v-stage>
      </v-flex>
      <v-flex xs4>
        <v-text-field
              label="Barcode"
              v-model="barcode"
              required
            >
            </v-text-field>
            <v-text-field
              label="Name"
              v-model="name"
              required
            >
            </v-text-field>
            <v-text-field
                label="Price"
                v-model="price"
                required
              >
              </v-text-field>
        <p>{{readText}}</p>
      </v-flex>
    </v-layout>
  </v-container>  
</template>

<script>
  export default {
    name: 'invoice',
    data() {
      return {
        tempResult: null,
        currency: 'CHF',
        readText: '',
        invoiceNumber: '',
        invoiceDate: '',
        invoiceSupplier: '',
        invoiceTotal: '',
        invoiceItems: [],
        barcode: '',
        name: '',
        price: '',
        sample: require('../assets/ReceiptSwiss.jpg'),
        image: null,
        scaleWidth: 1,
        scaleHeight: 1,
        configKonva: {
          width: 650,
          height: 750
        }
      }
    },
    created() {
      const image = new window.Image()
      image.src = require('../assets/ReceiptSwiss.jpg')
      image.onload = () => {
        this.image = image
        var orgWidth = image.width
        var orgHeight = image.height
        this.scaleWidth = 550 / orgWidth
        this.scaleHeight = 750 / orgHeight
        // irgendwie distortion berechnen und beruecksitigen
        var yoda = new Konva.Image({
          x: 0,
          y: 0,
          image: image,
          width: 550,
          height: 750
        })

        var stage = this.$refs.stage.getStage()
        var layer = new Konva.Layer()
        layer.add(yoda)
        stage.add(layer)
      }
    },
    methods: {
      initialVarFill: function() {
        var result = JSON.parse(localStorage.readFile) 
        var counter = 0
        var vm = this
        console.log(result)
        result.forEach(line => {
          // console.log(line)
          var strSomething = line['text']
          if(strSomething.includes(vm.currency)) {
            if(strSomething.includes("Total")) {
              // sum
            }else{
              //var x0 = Math.min(line.bbox.x0 * vm.scaleWidth, 50)
              var y0 = line.bbox.y0 * vm.scaleHeight
              //var x1 = Math.min(line.bbox.x1 * vm.scaleWidth, 550)
              var y1 = line.bbox.y1 * vm.scaleHeight
             // var w = Math.max((parseInt(x1) - parseInt(x0)), 500)
              var h = Math.max((parseInt(y1) - parseInt(y0)), 20)
              
              var pos = strSomething.indexOf(vm.currency)
              var tempAmount = strSomething.substring(0, pos)
              console.log("everything before curr " + tempAmount)
              tempAmount = tempAmount.match(/[+-]?\d+(?:\.\d+)?/g)
              tempAmount = tempAmount.replace(",",".")
              pos = tempAmount.lastIndexOf(".")
              //var pos2 = tempAmount
              console.log("now only number stuff " + tempAmount)

              var itemData = {
                x0: 550,
                y0: y0 + (h/2),
                name: strSomething,
                display: 'blue',
                price: tempAmount,
                barcode: ''
              }
              console.log(strSomething)
              vm.invoiceItems.push(itemData)
              counter = counter + 1
            }
          }
        })
      },
      drawIcons: function() {
        var stage = this.$refs.stage.getStage()
        stage.find('.groups').remove()
        var layer = new Konva.Layer({
          name: 'groups'
        })
        var counter = 0
        var vm = this
        this.invoiceItems.forEach(item => {
          var group = new Konva.Group({
            id: 'counter' + counter,
            draggable: true
          })
          // price line
          
          var rect = new Konva.Text({
            x: item.x0,
            y: item.y0,
            width: 80,
            height: 20,
            fill: 'gray',
            stroke: 'blue',
            strokeWidth: 2,
            text: item.price
          })
          rect.on('dragend', function() {
            vm.invoiceItems[counter]['x0'] = this.x
            vm.invoiceItems[counter]['y0'] = this.y
          })
          
          var checkCircle = new Konva.Circle({
            x: item.x0 + 80 + 10,
            y: item.y0 + 20,
            radius: 8,
            fill: item.display,
            stroke: 'black',
            strokeWidth: 1
          })
          checkCircle.on('click', function () {
            vm.invoiceItems[counter].display = 'green'
            vm.drawIcons()
          })
          
          var deleteCircle = new Konva.Circle({
            x: item.x0 + 80 + 25,
            y: item.y0 + 20,
            radius: 8,
            fill: 'red',
            stroke: 'black',
            strokeWidth: 1
          })
          deleteCircle.on('click', function () {
            vm.invoiceItems[counter] = null
            vm.drawIcons()
          })
          
          group.add(rect)
          group.add(checkCircle)
          group.add(deleteCircle)
          layer.add(group)
          counter = counter + 1
        })
        stage.add(layer)
      },
      startScan: function() {
        this.readText = "reading ..."
        //var vm = this
        Tesseract.recognize(this.sample)
        .then(function(result){
          var cache = []
          result.lines.forEach(line => {
            var data = {
              text: line['text'],
              confidence: line['confidence'],
              bbox: line.bbox,
              baseline: line.baseline
            }
            cache.push(data)
          })
          localStorage.readFile = JSON.stringify(cache)
        })
      }
    }
  }
</script>
