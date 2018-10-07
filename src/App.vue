<template>
  <div id="app">
    <line-chart v-bind="lineChartProps"></line-chart>
  </div>
</template>

<script>
import LineChart from "./components/LineChart";

export default {
  name: "App",
  components: {
    LineChart
  },computed:{
        dayData() {
      return [
        {
          name: '2 times',
          values: [
            {
              date: '14/09/2018',
              value: '18'
            },
            {
              date: '15/09/2018',
              value: '9'
            },
            {
              date: '16/09/2018',
              value: '4'
            },
            {
              date: '18/09/2018',
              value: '5'
            }
          ]
        },
        {
          name: '1 time',
          values: [
            {
              date: '14/09/2018',
              value: '13'
            },
            {
              date: '15/09/2018',
              value: '14'
            },
            {
              date: '16/09/2018',
              value: '6'
            },
            {
              date: '18/09/2018',
              value: '14'
            }
          ]
        },
        {
          name: '4 - 5 times',
          values: [
            {
              date: '14/09/2018',
              value: '5'
            },
            {
              date: '15/09/2018',
              value: '4'
            },
            {
              date: '18/09/2018',
              value: '1'
            }
          ]
        },
        {
          name: '6 - 8 times',
          values: [
            {
              date: '14/09/2018',
              value: '3'
            }
          ]
        },
        {
          name: 'more than 8 times',
          values: [
            {
              date: '14/09/2018',
              value: '2'
            },
            {
              date: '18/09/2018',
              value: '2'
            }
          ]
        },
        {
          name: '3 times',
          values: [
            {
              date: '14/09/2018',
              value: '11'
            },
            {
              date: '15/09/2018',
              value: '3'
            },
            {
              date: '16/09/2018',
              value: '3'
            },
            {
              date: '18/09/2018',
              value: '4'
            }
          ]
        }
      ];
    },
    lineChartProps(){
      return {
                xAxisLabel: 'Timeline',
        yAxisLabel: 'Value',
        xKey: 'date',
        yKey: 'value',
        interval: 'day',
        data: this.dayData

      }
    }
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
