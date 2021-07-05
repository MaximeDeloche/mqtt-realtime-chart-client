<template>
    <div>
        <div class="container-fluid">
            <div class="row text-center">
                
                <div class="col-xs-12 col-sm-8 col-sm-offset-2">
                    
                    <!-- Panel div start -->
                    <div class="panel panel-primary">
                        <div class="panel-heading">
                            <h3>Stock market prediction</h3>
                        </div>
                        <div class="panel-body">
                            <!-- Chart container -->
                            <div id="chart_container" >
                                <div id="y_axis"></div>
                                <div id="demo_chart" ref="panel"></div>
                            </div>
                            <!-- End of chart container -->
                        </div>
                        <div class="panel-footer">
            <div class="row">
                <div class="col-sm-6 text-left">
                <h4><b>
                    <span style="color: #254661">Real prices</span> &
                    <span style="color: #00a65e">Prediction</span>
                </b></h4>
                </div>
                <div class="col-sm-6 text-right">
                <h4>
                    {{connStatus}} 
                    <span class="dot" v-bind:class="{ green: connStatus == 'Connected'}"></span>
                </h4>
                </div>
                        </div>
                        </div>
                    </div>
                    <!-- Panel div end -->
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import io from 'socket.io-client'
    import Rickshaw from 'rickshaw'
    import 'rickshaw/rickshaw.min.css'
    import 'bootstrap/dist/css/bootstrap.css'
    var url = new URL(window.location);
    url.port = parseInt(url.port) + 1;
    var socket = io.connect(url.toString());
    var magnitudeChart;

    export default {
        name: 'home',
        data() {
            return {
        currentIndex: 0,
        maxLength: 100,
                updateInterval: 200,
                connStatus: "Disconnected",
            }
        },
        mounted() {
            this.initChart();
            this.openSocketListeners();
        },
        methods: {
            /* Rickshaw.js initialization */
            initChart() {
                magnitudeChart = new Rickshaw.Graph({
                    element: document.querySelector("#demo_chart"),
                    width: "500",
                    height: "180",
                    renderer: "line",
                    min: 50,
                    max: 100,
                    series: [
                        {
                            name: "reals",
                            data: [],
                            color: "#254661"
                        },
                        {
                            name: "predictions",
                            data: [],
                            color: "#00a65e"
                        }
                    ]
                    });

                var y_axis = new Rickshaw.Graph.Axis.Y({
                    graph: magnitudeChart,
                    orientation: 'left',
                    tickFormat: function(y) {
                        return y.toFixed(1);
                    },
                    ticks: 5,
                    element: document.getElementById('y_axis'),
                });

                this.resizeChart(magnitudeChart);

                window.addEventListener('resize', () => {
                    this.resizeChart(magnitudeChart)
                });

                // first series contains real prices, the second one the predicted next steps
                this.realPrices = magnitudeChart.series[0];
                this.predPrices = magnitudeChart.series[1];

            },


            resizeChart(chart) {
                chart.configure({
                    width: this.$refs.panel.clientWidth,
                });
                chart.render();
            },


            /* Insert received datapoints into the chart */
            insertDatapoints(message, chart) {
                // message contains:
                // {price: new real price from the last second,
                //  predictions: [list of predicted next prices]}

                // Step 1: update real prices
                // add new real price (with current index) to the list
                this.realPrices.data.push({x: this.currentIndex, y: message.price});
                // pop oldest element(s) if list reached max length
                while(this.realPrices.data.length > this.maxLength) {
                    this.realPrices.data.shift();
                }
                this.currentIndex ++; // increment current index for the next real price

                // Step 2: replace predicted prices
                var predIndex = this.currentIndex;
                // map predictions to indexes that follow the current index ('future' indexes)
                this.predPrices.data = message.predictions.map(elt => ({"x": predIndex++, "y": elt}));
                // prepend the last real price for the junction of the 2 plots
                this.predPrices.data.unshift(this.realPrices.data.slice(-1)[0]);

                chart.render();
            },


            openSocketListeners() {
                socket.on('connect', () => {
                    this.connStatus = "Connected";
                });

                socket.on('disconnect', () => {
                    this.connStatus = "Disconnected";
                });

                /* Update chart when price is received */
                socket.on('priceData', (message) => {
                    this.insertDatapoints(message, magnitudeChart);
                });
            }
        }
    }

</script>

<style scoped>
    #chart_container {
        padding-right: 40px;
        padding-bottom: 20px;
        margin-top: 20px;
        position: relative;
    }
    
    #demo_chart {
        position: relative;
        left: 40px;
    }
    
    #y_axis {
        position: absolute;
        top: 0;
        bottom: 0;
        width: 40px;
    }
    
    .footy {
        position: relative;
        width: 100%;
        margin-top: 50px;
        height: 60px;
        opacity: 0.2;
    }
    
    .glyphicon {
        color: #8E44AD;
        font-weight: bold;
    }
    .dot {
      height: 13px;
      width: 13px;
      background-color: #b51f1f;
      border-radius: 50%;
      display: inline-block;
    }
    .green {
      background-color: #099109;
    }

</style>
