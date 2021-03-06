<template>
  <div id="app">
    <div id="search">
      <input
        id="location-input"
        type="text"
        ref="input"
        placeholder="Location?"
        @keyup.enter="organizeAllDetails"
      />
      <button id="search-btn" @click="organizeAllDetails">
        <img src="./assets/Search.svg" width="24" height="24" />
      </button>
    </div>
    <div id="info">
      <div class="wrapper-left">
        <div id="current-weather">
          {{ currentWeather.temp }}
          <span>°C</span>
        </div>
        <div id="weather-desc">{{ currentWeather.summary }}</div>
        <div class="temp-max-min">
          <div class="max-desc">
            <div id="max-detail">
              <i>▲</i>
              {{ currentWeather.todayHighLow.todayTempHigh }}
              <span>°C</span>
            </div>
            <div id="max-summary">
              at {{ currentWeather.todayHighLow.todayTempHighTime }}
            </div>
          </div>
          <div class="min-desc">
            <div id="min-detail">
              <i>▼</i>
              {{ currentWeather.todayHighLow.todayTempLow }}
              <span>°C</span>
            </div>
            <div id="min-summary">
              at {{ currentWeather.todayHighLow.todayTempLowTime }}
            </div>
          </div>
        </div>
      </div>
      <div class="wrapper-right">
        <div class="date-time-info">
          <div id="date-desc">
            <img src="./assets/calendar.svg" width="20" height="20" />
            {{ currentWeather.time }}
          </div>
        </div>
        <div class="location-info">
          <div id="location-desc">
            <img
              src="./assets/location.svg"
              width="10.83"
              height="15.83"
              style="opacity: 0.9;"
            />
            {{ currentWeather.full_location }}
            <div id="location-detail" class="mt-1">
              Lat: {{ currentWeather.formatted_lat }}
              <br />
              Long: {{ currentWeather.formatted_long }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <dashboard-content
      :highlights="highlights"
      :tempVar="tempVar"
    ></dashboard-content>
  </div>
</template>

<script>
import Content from "./components/Content.vue";

export default {
  name: "app",
  components: {
    "dashboard-content": Content
  },
  data() {
    return {
      weatherDetails: false,
      location: "", // raw location from input
      lat: "", // raw latitude from google maps api response
      long: "", // raw longitude from google maps api response
      completeWeatherApi: "", // weather api string with lat and long
      rawWeatherData: "", // raw response from weather api
      currentWeather: {
        full_location: "", // for full address
        formatted_lat: "", // for N/S
        formatted_long: "", // for E/W
        time: "",
        temp: "",
        todayHighLow: {
          todayTempHigh: "",
          todayTempHighTime: "",
          todayTempLow: "",
          todayTempLowTime: ""
        },
        summary: "",
        possibility: ""
      },
      tempVar: {
        tempToday: [
          // gets added dynamically by this.getSetHourlyTempInfoToday()
        ]
      },
      highlights: {
        uvIndex: "",
        visibility: "",
        windStatus: {
          windSpeed: "",
          windDirection: "",
          derivedWindDirection: ""
        }
      }
    };
  },
  methods: {
    convertToTitleCase: function(str) {
      str = str.toLowerCase().split(" ");
      for (var i = 0; i < str.length; i++) {
        str[i] = str[i].charAt(0).toUpperCase() + str[i].slice(1);
      }
      return str.join(" ");
    },

    formatPossibility: function(str) {
      str = str.toLowerCase().split("-");
      for (var i = 0; i < str.length; i++) {
        str[i] = str[i].charAt(0).toUpperCase() + str[i].slice(1);
      }
      return str.join(" ");
    },

    unixToHuman: function(timezone, timestamp) {
      /* READ THIS BEFORE JUDGING & DEBUGGING
     For any location beyond the arctic circle and the
     antarctic circle, the goddamn weather api does not return certain
     keys/values in each of this.rawWeatherData.daily.data[some_array_index].
     Due to this, console throws up an error.
     The code is correct, the problem is with the API.
     May be later on I will add some padding to tackle missing values.
     */
      var moment = require("moment-timezone"); // for handling date & time
      var decipher = new Date(timestamp * 1000);
      var human = moment(decipher)
        .tz(timezone)
        .format("llll");
      var timeArray = human.split(" ");
      var timeNumeral = timeArray[4];
      var timeSuffix = timeArray[5];
      var justTime = timeNumeral + " " + timeSuffix;
      var monthDateArray = human.split(",");
      var monthDate = monthDateArray[1].trim();
      return {
        fullTime: human,
        onlyTime: justTime,
        onlyMonthDate: monthDate
      };
    },

    fahToCel: function(tempInFahrenheit) {
      var tempInCelcius = Math.round((5 / 9) * (tempInFahrenheit - 32));
      return tempInCelcius;
    },

    milibarToKiloPascal: function(pressureInMilibar) {
      var pressureInKPA = pressureInMilibar * 0.1;
      return Math.round(pressureInKPA);
    },

    mileToKilometer: function(miles) {
      var kilometer = miles * 1.60934;
      return Math.round(kilometer);
    },

    deriveWindDir: function(windDir) {
      var wind_directions_array = [
        { minVal: 0, maxVal: 30, direction: "N" },
        { minVal: 31, maxVal: 45, direction: "NNE" },
        { minVal: 46, maxVal: 75, direction: "NE" },
        { minVal: 76, maxVal: 90, direction: "ENE" },
        { minVal: 91, maxVal: 120, direction: "E" },
        { minVal: 121, maxVal: 135, direction: "ESE" },
        { minVal: 136, maxVal: 165, direction: "SE" },
        { minVal: 166, maxVal: 180, direction: "SSE" },
        { minVal: 181, maxVal: 210, direction: "S" },
        { minVal: 211, maxVal: 225, direction: "SSW" },
        { minVal: 226, maxVal: 255, direction: "SW" },
        { minVal: 256, maxVal: 270, direction: "WSW" },
        { minVal: 271, maxVal: 300, direction: "W" },
        { minVal: 301, maxVal: 315, direction: "WNW" },
        { minVal: 316, maxVal: 345, direction: "NW" },
        { minVal: 346, maxVal: 360, direction: "NNW" }
      ];
      var wind_direction = "";
      for (var i = 0; i < wind_directions_array.length; i++) {
        if (
          windDir >= wind_directions_array[i].minVal &&
          windDir <= wind_directions_array[i].maxVal
        ) {
          wind_direction = wind_directions_array[i].direction;
        }
      }
      return wind_direction;
    },

    makeInputEmpty: function() {
      this.$refs.input.value = "";
    },

    makeTempVarTodayEmpty: function() {
      this.tempVar.tempToday = [];
    },

    detectEnterKeyPress: function() {
      var input = this.$refs.input;
      input.addEventListener("keyup", function(event) {
        event.preventDefault();
        var enterKeyCode = 13;
        if (event.keyCode === enterKeyCode) {
          this.setHitEnterKeyTrue();
        }
      });
    },

    locationEntered: function() {
      var input = this.$refs.input;
      if (input.value === "") {
        this.location = "New York";
      } else {
        this.location = this.convertToTitleCase(input.value);
      }
      this.makeInputEmpty();
      this.makeTempVarTodayEmpty();
    }
  },
  computed: {}
};
</script>

<style></style>
