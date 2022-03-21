<script>
import { Point, Pub } from "./Objects.vue";
export default {
  emits: ["addPub"],
  data: function () {
    return {
      pubs: [],
      originPosition: null,
      defaultPubPic: "",
      chosenArray: [],
      viewPubs: false,
      map: null,
      googleSearchService: null,
      googleGeoCoder: null,
      directionsRenderer: new google.maps.DirectionsRenderer({
        preserveViewport: true,
      }),
      directionsService: new google.maps.DirectionsService(),
      pubListVisible: true,
      theme: "minimized",
      divList: "show-potentials",
      nearyouTheme: "closed",
      chosenTheme: "closed",
      popupTheme: true,
      ifRating: true,
      ifPriceClass: true,
      ifImage: true,
      globalPopup: Pub,
      getNextPage: null,
      x: new Array(3),
      counter: 0,
      previousButton: document.querySelector("#previousButton"),
      nextButton: document.querySelector("#nextButton"),
      lastAddress: null,
    };
  },
  mounted: function () {
    previousButton.disabled = true;

    for (let i = 0; i < this.x.length; i++) {
      this.x[i] = new Array();
    }

    this.initMap();
    this.geoLocate();
    if (localStorage.getItem("chosenArray")) {
      try {
        this.chosenArray = JSON.parse(localStorage.getItem("chosenArray"));
      } catch (e) {
        localStorage.removeItem("chosenArray");
      }
    }
  },
  watch: {
    chosenArray: {
      handler(newChosenArray) {
        const parsed = JSON.stringify(newChosenArray);
        localStorage.setItem("chosenArray", parsed);
        console.log(this.chosenArray)
        console.log(this.newChosenArray)
      },
      deep: true,
    },
  },
  methods: {
    initMap() {
      this.map = new google.maps.Map(document.getElementById("map"), {
        // center: { lat: 57.70887, lng: 11.97456 },
        zoom: 14,
        disableDefaultUI: true,
        fullscreenControl: true,
      });

      this.directionsRenderer.setMap(this.map);
      this.googleSearchService = new google.maps.places.PlacesService(this.map);
      this.googleGeoCoder = new google.maps.Geocoder();
    },
    async searchByName() {
      let request = {
        address: this.placeName,
      };

      //Converts the seach string to a geolocation
      this.googleGeoCoder.geocode(request, callback);

      const self = this;
      function callback(results, status) {
        if (status == "OK") {
          const locationLatLong = results[0].geometry.location;
          self.searchByGeoLocation(locationLatLong);
          document.querySelector(".switch-button").textContent =
            "Pubs near " + self.placeName;
          self.resetPageButtons();
          //   self.lastAddress = self.placeName;
          //   console.log(self.lastAddress)
        } else {
          self.notifyUser("No place found");
          //   self.placeName = self.lastAddress;
          //   console.log(self.lastAddress)
        }
      }
    },
    async searchByGeoLocation(geoLocation) {
      let request = {
        radius: "3000",
        location: geoLocation,
        type: "bar",
      };

      this.googleSearchService.textSearch(request, callback);

      const self = this;
      function callback(results, status, pageToken) {
        self.pubs = [];

        if (status == google.maps.places.PlacesServiceStatus.OK) {
          for (let i = 0; i < results.length; i++) {
            const point = new Point(
              results[i].geometry.location.lat(),
              results[i].geometry.location.lng()
            );
            const photo =
              "photos" in results[i]
                ? results[i].photos[0].getUrl({ maxHeight: 1000 })
                : self.defaultPubPic;

            self.pubs.push(
              new Pub(
                point,
                results[i].place_id,
                results[i].name,
                photo,
                results[i].rating,
                results[i].price_level,
                results[i].user_ratings_total,
                results[i].formatted_address
              )
            );
          }

          if (pageToken && pageToken.hasNextPage) {
            self.getNextPage = () => {
              // Note: nextPage will call the same handler function as the initial call
              pageToken.nextPage();

              self.counter++;
              console.log(self.counter);
            };
          } else {
            self.getNextPage = null;
          }
          self.nearYouButton();
        }

        //Stores each page results in an array. Store each array in an array.
        for (let i = 0; i < self.pubs.length; i++) {
          self.x[self.counter].push(self.pubs[i]);
        }

        console.log(self.x);
      }
      this.moveMapTo(geoLocation);
    },
    moveMapTo(location) {
      this.map.panTo(location);
    },
    addPub(pub) {
      for (let p of this.chosenArray) {
        if (p.name === pub.name) {
          this.notifyUser("Pub already added");
          return;
        }
      }
      this.chosenArray.push(pub);
      this.notifyUser("Pub added to crawl list");
    },
    removePub(pub) {
      for (let i = 0; i < this.chosenArray.length; i++) {
        if (this.chosenArray[i] === pub) {
          this.chosenArray.splice(i, 1);
        }
      }
    },
    async calculateAndDispalyRoutes(optimizeRoute = false) {
      const waypts = [];

      for (let i = 1; i < this.chosenArray.length - 1; i++) {
        waypts.push({
          location: { placeId: this.chosenArray[i].googleId },
          stopover: true,
        });
      }
      const self = this;
      this.directionsService
        .route({
          origin: { placeId: this.chosenArray[0].googleId },
          destination: {
            placeId: this.chosenArray[this.chosenArray.length - 1].googleId,
          },
          waypoints: waypts,
          optimizeWaypoints: optimizeRoute,
          travelMode: google.maps.TravelMode["WALKING"],
        })
        .then((response) => {
          if (optimizeRoute) {
            for (let i = 1; i < response.geocoded_waypoints.length - 1; i++) {
              let from = self.chosenArray.findIndex(
                (elem) =>
                  elem.googleId === response.geocoded_waypoints[i].place_id
              );
              self.moveItem(from, i);
            }
          }
          for (let i = 0; i < response.routes[0].legs.length; i++) {
            response.routes[0].legs[i].end_address =
              self.chosenArray[i + 1].name;
            response.routes[0].legs[i].start_address = self.chosenArray[i].name;
            self.chosenArray[i + 1].distance =
              response.routes[0].legs[i].distance.text;
          }

          this.directionsRenderer.setDirections(response);
        });
    },
    geoLocate() {
      let self = this;

      function success(position) {
        position = new Point(
          position.coords.latitude,
          position.coords.longitude
        );
        self.originPosition = position;
        self.moveMapTo(position);
        self.searchByGeoLocation(position);
      }

      function error() {
        self.notifyUser("Unable to retrieve your location");
      }

      if (!navigator.geolocation) {
        notifyUser("Geolocation is not supported by your browser");
      } else {
        navigator.geolocation.getCurrentPosition(success, error);
        document.querySelector(".switch-button").textContent = "Pubs near you";
        this.resetPageButtons();
      }
    },
    showAndHidePubs() {
      if (this.pubListVisible === true) {
        document.querySelector("#show-hide").textContent = "HIDE";
        this.theme = "maximized";
        this.pubListVisible = false;
      } else {
        document.querySelector("#show-hide").textContent = "SHOW";

        this.theme = "minimized";
        this.pubListVisible = true;
      }
    },
    forceShowPubs() {
      document.querySelector("#show-hide").textContent = "HIDE";
      this.theme = "maximized";
      this.pubListVisible = false;
    },
    nearYouButton() {
      this.forceShowPubs();
      this.divList = "show-potentials";
      document.querySelector("#response-list").classList.add("selected-list");
      document.querySelector("#chosen-list").classList.remove("selected-list");
      document
        .querySelectorAll("#chosen-list .menuButton")
        .forEach((e) => e.classList.add("hidden"));
      document
        .querySelectorAll("#response-list .menuButton")
        .forEach((e) => e.classList.remove("hidden"));

      this.chosenTheme = "closed";
      this.nearyouTheme = "open";
    },
    chosenButton() {
      this.forceShowPubs();
      this.divList = "show-chosenones";
      document
        .querySelector("#response-list")
        .classList.remove("selected-list");
      document.querySelector("#chosen-list").classList.add("selected-list");
      document
        .querySelectorAll(".menuButton")
        .forEach((e) => e.classList.remove("hidden"));
      document
        .querySelectorAll("#response-list .menuButton")
        .forEach((e) => e.classList.add("hidden"));

      this.chosenTheme = "open";
      this.nearyouTheme = "closed";
    },
    notifyUser(message) {
      const element = document.querySelector("#notification");
      element.classList.add("animate");
      element.textContent = message;

      setTimeout(function () {
        element.classList.remove("animate");
        element.textContent = "";
      }, 3000);
    },
    showPopup(result) {
      this.globalPopup = result;
      this.popupTheme = false;

      if (result.userRatings == 0) {
        this.ifRating = false;
      }

      if (result.priceClass == undefined) {
        this.ifPriceClass = false;
      }

      if (result.imgSrc == "") {
        this.ifImage = false;
      }
    },
    hidePopup() {
      this.popupTheme = true;
      this.ifRating = true;
      this.ifPriceClass = true;
      this.ifImage = true;
    },
    moveItem(from, to) {
      const f = this.chosenArray.splice(from, 1)[0];
      this.chosenArray.splice(to, 0, f);
    },
    moveUp(pub) {
      const pos = this.chosenArray.indexOf(pub);
      if (pos == 0) return;
      this.moveItem(pos, pos - 1);
    },
    moveDown(pub) {
      const pos = this.chosenArray.indexOf(pub);
      if (pos === this.chosenArray.length) return;
      this.moveItem(pos, pos + 1);
    },
    previousPage() {
      this.counter--;
      console.log(this.counter);

      if (this.counter < 2) {
        nextButton.disabled = false;
      } else {
        nextButton.disabled = true;
      }

      if (this.counter > 0) {
        previousButton.disabled = false;
      } else {
        previousButton.disabled = true;
      }
    },
    nextPage() {
      if (this.x[this.counter + 1].length > 0) {
        this.counter++;
      } else {
        if (this.getNextPage) {
          this.getNextPage();
        }
      }
      if (this.counter > 0) {
        previousButton.disabled = false;
      } else {
        previousButton.disabled = true;
      }

      if (this.counter < 2) {
        nextButton.disabled = false;
      } else {
        nextButton.disabled = true;
      }
    },
    resetPageButtons() {
      console.log(this.x);
      this.counter = 0;

      for (let i = 0; i < this.x.length; i++) {
        this.x[i] = [];
      }

      nextButton.disabled = false;
      previousButton.disabled = true;
      console.log(this.x);
    },
  },
};
</script>

<template>
  <section id="map-container">
    <!-- The visual map from Googles Maps JavaScript API displays on "map"-div -->
    <div id="map"></div>
  </section>
  <section id="search-section">
    <div id="search-box">
      <form id="search-form">
        <input
          id="search-field"
          type="search"
          placeholder="Find you target town"
          v-model="placeName"
        />
        <button
          @click.prevent="searchByName()"
          id="search-button"
          class="searchbar-button material-icons md48 beer-colored"
        >
          sports_bar
        </button>
        <button
          @click.prevent="geoLocate()"
          class="searchbar-button material-icons md24 beer-colored"
        >
          my_location
        </button>
      </form>
    </div>
    <div id="notification"></div>
  </section>
  <section v-if="!popupTheme" id="popup-section">
    <div class="popup">
      <button class="popup-close-button" @click="hidePopup(result)">✖</button>
      <p>
        <strong>{{ globalPopup.name }}</strong>
      </p>
      <p>{{ globalPopup.address }}</p>
      <div v-if="!ifPriceClass">No price class...</div>
      <div v-if="ifPriceClass" class="starDiv">
        Price class:
        <div
          class="price-class-a"
          v-for="index in Math.round(globalPopup.priceClass)"
          :key="index"
        >
          <strong>$</strong>
        </div>
        <div
          v-for="index in 5 - Math.round(globalPopup.priceClass)"
          :key="index"
          class="price-class-b"
        >
          <strong>$</strong>
        </div>
      </div>
      <hr />

      <div v-if="!ifRating">No rating...</div>
      <div v-if="ifRating" class="starDiv">
        Rating:
        <div v-for="index in Math.floor(globalPopup.rating)" :key="index">
          <span class="fa fa-star checked"></span>
        </div>
        <div v-for="index in 5 - Math.floor(globalPopup.rating)" :key="index">
          <span id="star-not-checked" class="fa fa-star"></span>
        </div>
      </div>

      <p>Number of ratings: {{ globalPopup.userRatings }}</p>
      <img v-if="ifImage" :src="globalPopup.imgSrc" />
      <img
        v-if="!ifImage"
        src="https://as2.ftcdn.net/v2/jpg/02/55/54/87/1000_F_255548787_NN93IpmZ29kRmNfy8OYWvSqLYSm1VCTj.jpg"
      />
    </div>
  </section>
  <section id="list-section" :class="theme">
    <div id="show-hide" class="beer-colored" @click="showAndHidePubs()">
      SHOW
    </div>
    <div id="div-list" :class="divList">
      <div id="response-list" class="list">
        <button class="switch-button" @click="nearYouButton(this)">
          Pubs near you
        </button>
        <ul>
          <li
            v-for="result in x[counter]"
            :key="result.name"
            :class="nearyouTheme"
            aria-haspopup="true"
          >
            <div class="icon-name">
              <i
                v-if="chosenArray.includes(result)"
                class="searchbar-button material-icons md48 beer-colored"
                >done</i
              >
              <p class="li-name" @click="showPopup(result)">
                {{ result.name }}
              </p>
            </div>

            <button class="add-remove-button" @click="addPub(result)">+</button>
          </li>
        </ul>
        <div>
          <button
            @click="previousPage()"
            id="previousButton"
            class="menuButton"
          >
            Previous
          </button>
          <button @click="nextPage()" id="nextButton" class="menuButton">
            Next
          </button>
        </div>
      </div>
      <div id="chosen-list" class="list">
        <div id="switch-buttons">
          <button id="switch-1" class="switch-button" @click="chosenButton()">
            Planned Crawl
          </button>
          <button
            id="switch-2"
            class="menuButton hidden"
            type="button"
            @click.prevent="calculateAndDispalyRoutes(false)"
          >
            Show Route
          </button>
          <button
            id="switch-3"
            class="menuButton hidden"
            type="button"
            @click.prevent="calculateAndDispalyRoutes(true)"
          >
            Optimize Route
          </button>
        </div>
        <ul>
          <li
            v-for="result in chosenArray"
            :key="result.name"
            :class="chosenTheme"
          >
            <p @click="showPopup(result)">{{ result.name }}</p>
            <div class="order-pubs">
              <span class="material-icons" @click="moveUp(result)"
                >expand_less</span
              >
              <span class="material-icons" @click="moveDown(result)"
                >expand_more</span
              >
            </div>
            <button class="add-remove-button" @click="removePub(result)">
              -
            </button>
          </li>
        </ul>
      </div>
    </div>
  </section>
</template>