<script lang="ts">

export class Point {
  lat: string;
  lng: string;

  constructor(lat: string, lng: string) {
    this.lat = lat;
    this.lng = lng;
  }
}

export class Pub {
  location: Point;
  googleId: string;
  name: string;
  imgSrc: string;
  rating: number;
  priceClass: number;
  userRatings: number;
  address: string;
  photos: [string];
  distance: string;

  constructor (location: Point, googleId: string, name: string, imgSrc, rating: number, priceClass: number, userRatings: number, address: string, photos: [string]) {
      this.location = location;
      this.googleId = googleId;
      this.name = name;
      this.imgSrc = imgSrc;
      this.rating = rating;
      this.priceClass = priceClass;
      this.userRatings = userRatings;
      this.address = address;
      this.photos = photos
  }
}

</script>