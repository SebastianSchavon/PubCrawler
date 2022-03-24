<script>
import { Point, Pub } from "./Objects.vue";
export default {
  data: function () {
    return {
      // Stores all the pub objects from the API response.
      pubs: [],
      // Stores all the pub objects chosen by the user.
      chosenArray: [],
      // Stores each page from API response.
      pageArray: new Array(3),

      // Variables below control which class to be active at listSection and divList
      listSectionTheme: "minimized",
      divListTheme: "show-potentials",

      // Boolean values below checks if the API response contains rating, price and image-values.
      ifRatingExists: true,
      ifPriceExists: true,
      ifImageExists: true,

      // Boolean values below controls v-if states on menuButtons and the popup div.
      menuButtonHidden: true,
      popupDivHidden: true,

      // Popup div which contains and displays a selected pub.
      globalPopup: Pub,

      // Stores the 'Next Page Token' from the API response.
      getNextPage: null,

      // Control which of the arrays in pageArray to display/manipulate.
      pageCounter: 0,

      // Show/Hide button textcontent
      showHide: "SHOW",
      
      // Used to control the textcontet of the HIDE/SHOW button.
      pubListVisible: true,

      // Google API objects below
      map: null,

      // Will contain googles PlacesService and Geocoder class after pageload
      googleSearchService: null,
      googleGeoCoder: null,

      // Renders the results from directionsService
      directionsRenderer: new window.google.maps.DirectionsRenderer({
        preserveViewport: true,
      }),
      // Communicates with googles Directions Service which receives direction requests and returns an efficient path.
      directionsService: new window.google.maps.DirectionsService(),
    };
  },
  mounted: function () {
    this.initMap();
    this.geoLocate();

    // Retrieves the list of chosen pubs from users localstorage.
    if (localStorage.getItem("chosenArray")) {
      try {
        let temp = JSON.parse(localStorage.getItem("chosenArray"));
        for (let i = 0; i < temp.length; i++) {
          this.chosenArray[i] = new Pub(
            temp[i].location,
            temp[i].googleId,
            temp[i].name,
            temp[i].imgSrc,
            temp[i].rating,
            temp[i].priceClass,
            temp[i].userRatings,
            temp[i].address
          );
        }
      } catch (e) {
        localStorage.removeItem("chosenArray");
      }
    }
  },
  watch: {
    // Stores the chosenArray on users localstorage
    chosenArray: {
      handler(newChosenArray) {
        const parsed = JSON.stringify(newChosenArray);
        localStorage.setItem("chosenArray", parsed);
      },
      deep: true,
    },
  },
  methods: {
    initMap() {
      // Initializes the visual map from Googles 'Maps Javascript API'
      this.map = new window.google.maps.Map(document.getElementById("map"), {
        center: { lat: 57.708870, lng: 11.974560 },
        zoom: 14,
        disableDefaultUI: true,
        fullscreenControl: true,
      });

      this.directionsRenderer.setMap(this.map);

      // PlacesService contains methods related to searching for places and retrieving details about a place.
      this.googleSearchService = new window.google.maps.places.PlacesService(this.map);

      // Geocoding is the process of converting addresses into geographic coordinates
      this.googleGeoCoder = new window.google.maps.Geocoder();
    },
    // Retrives coordinates from a user text-input search. Applies the cords to the GeoLocation method.
    async searchByName() {
      let request = {
        address: this.placeName,
      };

      // Converts the seach string to a geolocation
      this.googleGeoCoder.geocode(request, callback);

      const self = this;
      function callback(results, status) {
        if (status == "OK") {
          const locationLatLong = results[0].geometry.location;
          self.searchByGeoLocation(locationLatLong);

          document.querySelector("#response-title em").textContent =
            self.placeName;

          self.resetPageButtons();
        } else {
          self.notifyUser("No place found");
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

      /* The callback parameters is pre-declared by Google and contains the API response values.
      'result' contains the array with pubs, 'status' the response-message and 
      'pageToken' the string value to be used when requesting 20 more results */
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

            // Push each pub to array as pub objects
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

              self.pageCounter++;
            };
          } else {
            self.getNextPage = null;
          }
          self.nearYouButton();
        }

        // Stores each page results in an array. Store each array in an array.
        for (let i = 0; i < self.pubs.length; i++) {
          self.pageArray[self.pageCounter].push(self.pubs[i]);
        }
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
    // Displays the chosenArray pubs on the map with markers and route.
    async calculateAndDispalyRoutes(optimizeRoute = false) {
      const waypts = [];

      /* The route is between the first and the last element of the array as origin and destination. The remaning elements is set as 
      waypoints in between */
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

          // Changes each markers textcontent on the map. From marker-adress to the pub-name.
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
    removeRoute() {
      // Clear route from map
      this.directionsRenderer.set("directions", null);
    },
    // Find users location via Geolocation
    geoLocate() {
      let self = this;

      function success(position) {
        position = new Point(
          position.coords.latitude,
          position.coords.longitude
        );

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
        document.querySelector("#response-title em").textContent = "you";
        this.resetPageButtons();
      }
    },
    showAndHidePubs() {
      if (this.pubListVisible === true) {
        this.showHide = "HIDE";
        this.listSectionTheme = "maximized";
        this.pubListVisible = false;
      } else {
        document.querySelector("#show-hide").textContent = "SHOW";
        this.showHide = "SHOW";
        this.listSectionTheme = "minimized";
        this.pubListVisible = true;
      }
    },
    forceShowPubs() {
      this.showHide = "HIDE";
      this.listSectionTheme = "maximized";
      this.pubListVisible = false;
    },
    nearYouButton() {
      this.forceShowPubs();
      this.divListTheme = "show-potentials";
      document.querySelector("#response-list").classList.add("selected-list");
      document.querySelector("#chosen-list").classList.remove("selected-list");

      this.menuButtonHidden = false;
    },
    chosenButton() {
      this.menuButtonHidden = true;

      this.forceShowPubs();
      this.divListTheme = "show-chosenones";
      document
        .querySelector("#response-list")
        .classList.remove("selected-list");
      document.querySelector("#chosen-list").classList.add("selected-list");
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
      this.popupDivHidden = false;

      if (result.userRatings == 0) {
        this.ifRatingExists = false;
      }

      if (result.priceClass == undefined) {
        this.ifPriceExists = false;
      }

      if (result.imgSrc == undefined) {
        this.ifImageExists = false;
      }
    },
    hidePopup() {
      // Reset popup booleans
      this.popupDivHidden = true;
      this.ifRatingExists = true;
      this.ifPriceExists = true;
      this.ifImageExists = true;
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
    nextPage() {
      if (this.pageArray[this.pageCounter + 1].length > 0) {
        this.pageCounter++;
      } else {
        if (this.getNextPage) {
          this.getNextPage();
        }
      }
    },
    resetPageButtons() {
      this.pageCounter = 0;

      for (let i = 0; i < this.pageArray.length; i++) {
        this.pageArray[i] = [];
      }
    },
  },
};
</script>

<template>
  <section id="map-container">
    <!-- Div which is referenced in the API methods. Needs to be defined as "map" -->
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
          autocomplete="off"
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
  <section v-if="!popupDivHidden" id="popup-section">
    <div class="popup">
      <button class="popup-close-button" @click="hidePopup(result)">✖</button>
      <p>
        <strong>{{ globalPopup.name }}</strong>
      </p>
      <p>{{ globalPopup.address }}</p>
      <div v-if="!ifPriceExists">No price class...</div>
      <div v-if="ifPriceExists" class="starDiv">
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

      <div v-if="!ifRatingExists">No rating...</div>
      <div v-if="ifRatingExists" class="starDiv">
        Rating:
        <div v-for="index in Math.floor(globalPopup.rating)" :key="index">
          <span class="fa fa-star checked"></span>
        </div>
        <div v-for="index in 5 - Math.floor(globalPopup.rating)" :key="index">
          <span id="star-not-checked" class="fa fa-star"></span>
        </div>
      </div>

      <p>Number of ratings: {{ globalPopup.userRatings }}</p>
      <img v-if="ifImageExists" :src="globalPopup.imgSrc" />
      <img
        v-if="!ifImageExists"
        src="https://as2.ftcdn.net/v2/jpg/02/55/54/87/1000_F_255548787_NN93IpmZ29kRmNfy8OYWvSqLYSm1VCTj.jpg"
      />
    </div>
  </section>
  <section id="list-section" :class="listSectionTheme">
    <div id="show-hide" class="beer-colored" @click="showAndHidePubs()">
      {{ showHide }}
    </div>
    <div id="div-list" :class="divListTheme">
      <div id="response-list" class="list">
        <button
          id="response-title"
          class="switch-button"
          @click="nearYouButton(this)"
        >
          <p>
            Pubs near <em><b>you</b></em>
          </p>
        </button>
        <ul v-if="!menuButtonHidden">
          <li v-for="result in pageArray[pageCounter]" :key="result.name">
            <div class="icon-name">
              <i
                v-if="chosenArray.some((x) => x.name == result.name)"
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
        <div v-if="listSectionTheme == 'maximized'">
          <button
            v-if="!menuButtonHidden"
            v-bind:disabled="pageCounter == 0"
            @click="pageCounter--"
            id="previousButton"
            class="menuButton"
          >
            <span class="material-icons-outlined"> chevron_left </span>
          </button>
          <button
            v-if="!menuButtonHidden"
            @click="nextPage()"
            v-bind:disabled="
              !pageArray[pageCounter + 1] ||
              (pageArray[pageCounter + 1]?.length < 1 && !getNextPage)
            "
            id="nextButton"
            class="menuButton"
          >
            <span class="material-icons-outlined"> chevron_right </span>
          </button>
        </div>
      </div>
      <div id="chosen-list" class="list">
        <div id="switch-buttons">
          <button id="switch-1" class="switch-button" @click="chosenButton()">
            <p>Planned Crawl</p>
          </button>
        </div>
        <ul v-if="menuButtonHidden">
          <li v-for="result in chosenArray" :key="result.name">
            <p @click="showPopup(result)">{{ result.name }}</p>
            <div class="order-pubs">
              <span class="material-icons" @click="moveUp(result)"
                >expand_less</span
              >
              <span class="material-icons" @click="moveDown(result)"
                >expand_more</span
              >
              <div class="end-start"></div>
            </div>
            <button class="add-remove-button" @click="removePub(result)">
              -
            </button>
          </li>
        </ul>
        <div v-if="listSectionTheme == 'maximized'">
          <button
            v-bind:disabled="chosenArray?.length == 0"
            v-if="menuButtonHidden"
            id="switch-2"
            class="menuButton"
            type="button"
            @click.prevent="calculateAndDispalyRoutes(false)"
            title="Display pub crawl route"
          >
            Show Route
          </button>
          <button
            v-bind:disabled="chosenArray?.length == 0"
            v-if="menuButtonHidden"
            id="switch-3"
            class="menuButton"
            type="button"
            @click.prevent="calculateAndDispalyRoutes(true)"
            title="Optimize walking route between the pubs which is not start or end-pubs"
          >
            Optimize
          </button>
          <button
            v-bind:disabled="chosenArray?.length == 0"
            v-if="menuButtonHidden"
            id="switch-4"
            class="menuButton"
            @click="removeRoute()"
            title="Remove displayed route from map"
          >
            Remove
          </button>
        </div>
      </div>
    </div>
  </section>
</template>