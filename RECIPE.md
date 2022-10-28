# Measuring Request/Response Times with Hotwire

## Problem

Picture this scenario: you have a rails app, and it's working fine with zero bugs but you are still not sure of how fast and responsive it is. You would like to know the time a request takes from the time a user clicks on a button to the time the screen changes in response to that action.

There are multiple commercial options available that will enable you to do that but at a cost(of both time and money). The question is: if you wanted a simple measurement of the time between a client request and server response in your app could you do it on your own with just the rails stack alone minus any external dependencies? And if so, how would you go about it?

Hopefully, by the end of this post, you'll have an idea.

## Solution

To kick things off we need to identify what kind of data/metrics we are interested in measuring and storing. In our case, these are the **client request** time which is when a user clicks on a button and the **client response** time which is when the user's browser changes in response to the server response. We also need the difference between those two times which is the crucial [**ping time**](https://www.activexperts.com/network-component/tutorials/ping/). These metrics can be represented as attributes in an aptly named **NetworkTime** model and its corresponding migration shown below.

```
app/models/network_time.rb

class NetworkTime < ApplicationRecord
  validates :client_request_time, :client_response_time, :ping_time, presence: true

  after_create_commit -> { broadcast_prepend_to "network_times", partial: "network_times/network_time", locals: { network_times: self }, target: "network_times" }
end
```

```
app/db/migrate/....create_network_times.rb

class CreateNetworkTimes < ActiveRecord::Migration[7.0]
  def change
    create_table :network_times do |t|
      t.datetime :client_request_time, null: false
      t.datetime :client_response_time, null: false
      t.decimal :ping_time, null: false, precision: 10, scale: 2

      t.timestamps
    end
  end
end

```

One key thing to note here in our schema file is that the `client_request_time` and `client_response_time` attributes are stored in our database as DateTime fields which is self-explanatory but the `ping_time` is saved as a decimal field because it is a computed value.

Our model file contains the standard presence validations and a hotwire-specific `after_create_commit` callback method that runs every time a new NetworkTime instance is created. This method has a lambda function which broadcasts each newly created instance(in the form of a partial wrapped up in a turbo frame) to the target id section named `network_times` in the network time model's index view. The network times index view and partial are shown below.

```
apps/views/network_times/_network_time.html.erb

<%= turbo_frame_tag network_time do %>
  <div>
    <p>Client request time <%= network_time.client_request_time %></p>
    <p>Client response time <%= network_time.client_response_time %></p>
    <p>Ping time <%= network_time.ping_time %> ms </p>
  </div>
<% end %>
```

```
apps/views/network_times/index.html.erb

<main>
  <div data-controller="displays">
    <%= button_to "Start", network_times_path, data: { action: "displays#recurringPingTimes:prevent", "displays-request-time-value": "", "displays-response-time-value": "", "displays-ping-time-value": "" } %>
    <%= button_to "Stop", "#", data: { action: "displays#stopRecurringPingTimes:prevent"} %>
  </div>

  <%= turbo_stream_from "network_times" %>

  <%= turbo_frame_tag "network_times" do %>
    <%= render @network_times %>
  <% end %>
</main>
```

There's nothing special about the partial, it does the usual calls to model instance attributes and is eventually rendered but the index view is where the magic happens. In it, we have a `turbo_stream_from` tag that streams newly created network time model instances from the `network_times` stream and a `turbo_frame_tag` with the `network_times` id where all such instances will be prepended.

Right above that, however, are two `Start` and `Stop` buttons which power both the client and server actions on our app and this is where Stimulus JS comes into the mix. Both buttons are nested inside a div which maps to a stimulus controller and they both contain actions which prevent the default browser action and call specific methods in the controller. To put their operation into perspective the full stimulus controller is shown below.

```
app/javascripts/controllers/displays_controller.js

import { Controller } from "@hotwired/stimulus"
import { get, post } from "@rails/request.js"

// Connects to data-controller="displays"
export default class extends Controller {
  static values = { requestTime: String, responseTime: String, pingTime: String }

  setRequestTime(){
    this.requestTimeValue = new Date().toISOString()
  }

  calculatePingTime(){
    this.pingTimeValue = Date.parse(this.responseTimeValue) - Date.parse(this.requestTimeValue)
  }

  async pingTimeValueChanged(){
    let body = JSON.stringify({
        client_request_time: this.requestTimeValue,
        client_response_time: this.responseTimeValue,
        ping_time: this.pingTimeValue,
      })
    const postResponse = await post('http://localhost:3000/network_times',{
      body: body,
      responseKind: "turbo-stream"
    })
  }

  async getResponseTime(){
    this.setRequestTime()

    const getResponse = await get('http://localhost:3000/')
    if (getResponse.ok){
      this.responseTimeValue = new Date().toISOString()
      this.calculatePingTime()
    }
  }

  retrievePingTimes(){
    this.setRequestTime()
    this.getResponseTime()
  }

  recurringPingTimes(){
    this.refreshTimer = setInterval(() => {
      this.retrievePingTimes()
    }, 1000)
  }

  stopRecurringPingTimes() {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
    }
  }
}
```

To produce the final result of displaying our time metrics, our stimulus controller has to do 5 things:

- Make a request to our app and get the time the request was sent i.e. `client_request_time`.
- Receive the server response from our app and get the time the response was received i.e. `client_response_time`.
- Compute the difference between the two times i.e. `ping_time`.
- Post the time data back to our rails app.
- Repeat the process continuously per second until stopped.

Our stimulus controller's `getResponseTime()` async method is responsible for sending a `HTTP GET` request to our apps root page via the `rails/@request.js` library.  This method sets the client response time immediately via the `setRequestTime()` method and after a successful response(which is guaranteed because we are not doing anything complicated) it will proceed to set both the client response time and the ping time via the `calculatePingTime()` method. When our ping time value changes, it triggers a `pingTimeValueChanged()` callback which makes a post request of all network time values to our app.

For that request to succeed we'd need a corresponding controller and route which are shown below.

```
app/controllers/network_times_controller.rb

class NetworkTimesController < ApplicationController
  def index
    @network_times = NetworkTime.all
  end

  def create
    @network_time = NetworkTime.new(network_times_params)
    if @network_time.save
      respond_to do |format|
        format.turbo_stream
      end
    else
      head :internal_server_error
    end
  end

  private

  def network_times_params
    params.require(:network_time).permit(:client_request_time, :client_response_time, :ping_time)
  end
end
```

```
app/config/routes.rb

Rails.application.routes.draw do
  root "network_times#index"
  resources :network_times, only: [:index, :create]
end
```

One important thing, in order to match the recurring nature of an actual ping request, we need to automate this entire process to run after every second. This is why our `Start` button calls the `recurringPingTimes()` method which makes a call to the `retrievePingTimes()` method after every second via `setInterval` function and stores its timer id in the `refreshTimer` attribute. Doing so will ensure that every newly created network time instance is rendered on the page continuously until another click to the `Stop` button puts an end to the recurring calls. To put things into perspective this particular section of the displays controller is shown below.

```
app/javascripts/controllers/displays_controller.js
.......

retrievePingTimes(){
    this.setRequestTime()
    this.getResponseTime()
  }

  recurringPingTimes(){
    this.refreshTimer = setInterval(() => {
      this.retrievePingTimes()
    }, 1000)
  }

  stopRecurringPingTimes() {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
    }
  }

........
```

And with that, we have an inbuilt ping tool that displays recurring request, response and ping times with just one click of a button as seen below.

![Fly Ping Process demo](./fly.gif)

## Discussion (optional)

It's nice to see two separate languages(Ruby/Javascript) work together in tandem to achieve a common goal and that's what Hotwire brings to the table.

The patchy relationship Rails had with Javascript is all but gone as has been shown in this example. Here's to more fun and interactive apps written in the 'boring rails' way!
