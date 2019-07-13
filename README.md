# Real Estate Platform
Single-page, full stack web application **(SPA)** similar to Airbnb where users can view, book, and search for listings by location.

It utilizes **Ruby on Rails** with a **PostgreSQL** database on the back-end, and **React.js** and **Redux** on the front-end. 

## User Authentication
* Users can sign up or log in to use the application
* Users can also log in through a demo account

User credentials are securely hashed, salted, and stored as a password digest

```ruby
class User < ApplicationRecord
  validates :username, :session_token, uniqueness: true
  validates :username, presence: true
  validates :password, length: {minimum: 6, allow_nil: true}

  // ..
  
  attr_reader :password
  before_validation :ensure_session_token

  def self.find_by_credentials(username, password)
    @user = User.find_by(username: username)
    return nil unless @user
    @user.is_password?(password) ? @user : nil
  end

  def password=(password)
    @password = password
    self.password_digest = BCrypt::Password.create(password)
  end

  def is_password?(password)
    BCrypt::Password.new(self.password_digest).is_password?(password)
  end

  def ensure_session_token
    self.session_token ||= SecureRandom::urlsafe_base64
  end

  def reset_session_token!
    self.session_token = SecureRandom::urlsafe_base64
  end
end

```

## Listings
* Listings are displayed on the homepage
* Users are able to search for listings via Google Maps Places API

As a user moves the map around, the new bounds (coordinates) will get updated in realtime and send the correct listings from the backend (PostgreSQL database)

```ruby
class Listing < ApplicationRecord
  // ..
  
  def self.in_bounds(bounds)
    bounds = JSON.parse(bounds)

    self.where('lat < ?', bounds["northEast"]["lat"].to_f)
      .where('lat >?', bounds["southWest"]["lat"].to_f)
      .where('long < ?', bounds["northEast"]["lng"].to_f)
      .where('long > ?', bounds["southWest"]["lng"].to_f)
  end
  
  // ..
end
```
The Google Maps API is integrated into the appropriate React frontend components

```ruby
class ListingMap extends React.Component {
  constructor(props) {
    super(props);
    this.renderMap = this.renderMap.bind(this);
    this.handleMarkerClick = this.handleMarkerClick.bind(this);
  }
  
  // ..

  renderMap() {
    const mapOptions = {
      center: this.props.mapSearchCoords,
      zoom: 13,
      gestureHandling: "greedy"
    };

    this.map = new google.maps.Map(this.mapNode, mapOptions);
    this.MarkerManager = new MarkerManager(this.map, this.handleMarkerClick);

    this.registerListeners();
    this.MarkerManager.updateMarkers(this.props.listings);
  }

  registerListeners() {
    google.maps.event.addListener(this.map, 'idle', () => {
      const { north, south, east, west } = this.map.getBounds().toJSON();
      
      let bounds = {
        northEast: { lat: north, lng: east },
        southWest: { lat: south, lng: west }
      };

      this.props.updateFilter("bounds", bounds);
    });
    
    // ..
  }
```

## Bookings
* A logged in user is able to view his or her bookings
* A logged in user is able to make valid bookings on listings and delete any booking he or she made
