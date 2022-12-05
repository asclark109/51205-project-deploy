# team `TMACs`: Webay

# installation (backend+gateway)

`project-deploy` is a git repository that is sufficient for deploying the application. It contains scripts to clone the other repositories, build images based on those repositories, and orchestrate application startup (`docker-compose`).

1. git clone this repository:
```
$ git clone https://github.com/MPCS51205-TMACs/project-deploy
```

2. cd into `project-deploy`. there will be a scripts directory containing bash scripts. execute these scripts in order
```
$ . scripts/set-env-vars.sh
$ . scripts/clone-or-pull-repos.sh
$ . scripts/build-all-images.sh
```
The first script sets environment variables used in future steps. The second script clones the needed repositories. The third script will use the environment variables defined prior, which contain assumed information about paths to various files in the cloned repositories, and will build images from the repositories. This may take some time. If any errors are encountered running `build-all-images.sh`, simply try running it again. In the past this worked occasionally.

3. cd to the `project-deploy`. There is a `docker-compose` file there. Run `docker-compose up -d`. This may build a few more images on the first call to bring up the application.

The above steps should bring up the application.

### occasional bug / helpful to knows after calling docker-compose up -d (look for these)

1. after a call to `docker-compose up -d`, check the health of the `gateway` container by doing `docker logs -ft gateway`. Wait until the maven files have finished downloading. If the logs end with a "tomcat/petty started on port 8080", then it is good. If you encounter a build failure, check the changed files in the gateway repository (do a `git status` within the gateway repository). We noticed that in-between docker-compose files, some times additional configuration files will spawn that cause the build to fail. Delete the files if they appear and restart gateway by: deleting all newly appeared files and running `docker restart gateway`.

2. to see if the whole system is "up" look at the logs of the `auctions-service` container (`docker logs -ft auctions-service`). This container continuously tries to curl 3 other containers before executing its main function. So, you can watch these logs until the log activity comes to a slow.

3. after the auctions-service logs look "fine", do a `docker ps -a` to see if any containers have exited or are restarting. If not, the whole backend is up.

### interface to backend

The whole backend is now accessible by the gateway, which is exposed on localhost port 8080; that is, UI clients will send all requests to `http://localhost:8080/`, and the gateway will reach other containers as necessary.

# installation (UI)

we did not have enough time to dockerize the UI. `npm` must be downloaded locally.

1. git clone the UI repository:
```
$ git clone https://github.com/MPCS51205-TMACs/frontend
```

2. cd into the `frontend/frontend` folder. This is the directory from which calls will be made to start up the UI.

3. download node.js (`npm`): https://nodejs.org/en/download/

4. Now you, you should have an `npm`-like command you can use at the terminal. run the UI in a debug type mode (helpful for interactivity) by running
```
$ npm dev run
```

> note: you may encounter an error about not having a particular package. We did not end up using this package, but it is written as being imported somewhere in the code. We may not remember to delete this import statement by time of submission. So if it has not been deleted then first install the package by doing at the terminal: `npm install vue-splitpane`. the `npm dev run` should work thereafter.

The UI is accessed on https://localhost:3000/ likely.

# Architecture (technical)

client-server relationship between frontend and gateway docker container. microservice architecture of ~7 microservices, which each have an accompanying database container.

* `auctions-service`: golang/postgres (no frameworks). 
* `user-service`: java/postgres (+spring/boot).
* `item-service`: java/postgres (+spring/boot).
* `cam-service` (closed-auction-metrics): python/mongo (no frameworks).
* `shopping-cart-service`: python/mongo (no frameworks).
* `watchlist-service`: java/postgres (+spring/boot).
* `notification-service`: java/postgres (+spring/boot).

# Architecture (logical)

Among the 7 microservices, 3 stand out as major services: `user-service`, `item-service`, `auctions-service`, and 4 take on peripheral roles (`cam-service`, `shopping-cart-service`, `watchlist-service`, `notification-service`)

## `user-service`, `item-service`

`user-service` and `item-service` support major CRUD operations for users and items. A java-based spring/boot framework was used to streamline implementation of the common logic surrounding CRUD. This also helped ensure correctness and agility when changing schema (spring/boot allows users to create databases based on defined Java classes, so changes to domain model do not heavily burden the developer with schema changes at the database level). Postgres was chosen as a database because the schema surrounding users and items was well understood and unlikely to change much anyhow.

Both services publish a handful of messages to RabbitMQ `fanout` style. examples include (there are a handful more):
* `item.counterfeit`
* `item.create`
    * An example of reactivity: watchlist-service listens for new items to see if that new item will match the watchlist of any user in the system and if so will invoke the `notification-service` to notify that user.
* `user.create`
    * An example of reactivity: `notification-service` writes down the email of that user. That allows the `notification-service` to not have to talk to the `user-service` to get the user's contact information every time it wanted to alert that user. Consider how many network calls this would be during an active auction where many notifications need to be sent out. Also, it is not critical if the `notification-service` has slightly out-dated information about the user's contact details.
* `user.activation` (e.g. suspend or unsuspend user)
    * An example of reactivity: auctions-service listens to `user.activation` and `user.delete`, and it renders that user's bids for active auctions disabled--ensuring they cannot win any auctions henceforth. An event like `user.activation` indicating a user getting re-activated will re-activate his bids in any active auctions if there are any. `notification-service` may remember to not notify a user if they have suspended their account (so they don't get emails).

## `auctions-service`

`auctions-service` takes on a more autonomous role with lots of computation. It also has to manage many activities in parallel. To match the levels of computational work, we choose a more low-level language with performant built-in parallelism constructs--Golang. For now, we implemented the service with a __coarse-grained parallel implementation__ by putting a `mutex` lock around the service's interface. That is, we spawn multiple goroutines-each responsible to invoke the service at different circumstances, and the mutex lock ensures that there are no race conditions while managing these activities. `auctions-service` has a very small interface. It can be invoked directly through its RESTful API to create (submit) a new bid for an auction, or to cancel/stop/create an auction. 

* submitting of bids is done like so: the gateway sends a POST request to the `auctions-service` create a new Bid. To maximize throughput, the `auctions-service` simply stamps a time on the bid information and publishes it to a rabbitMQ exchange--which it also subscribes to. This activity is handled on one goroutine. Physically concurrently, a separate goroutine is subscribbed the bid creation exchange contained stamped bid data, and is processing those bids. The decision to publish the new bid information rather than process the bid when the request is received is to increase throughput. At auction end, many bids will flood the system, and so there must be a way for the system capture the many bids, and then process them at a rate it can handle. The auction system does this and has the ability to even process bids after the auction has ended (every bid that was published contains the time at which it was received, and the `Auction` domain object is capable of processing that bid based on the time it was received retroactively--i.e. the `Auction` is aware that its state is a function of time and thus can understand its state at any point in time). 
* creation and canceling of Auctions are done from the items-service. `items-service` synchronously invokes the `auctions-service` to create a new auction or cancel an existing one whenever a create item or delete item occurs (creation/deletion of an item succeeds if and only if an accompanied creation/cancellation of an auction is done).

In summary, the `auctions-service` has only 2 methods exposed to the gateway (excluding GET requests): `createBid()` and `stopAuction()`. `createBid()`
is dicussed above, and `stopAuction()` is a more powerful admin-like action. To differentiate `cancelAuction()` and `stopAuction()`: `cancelAuction()` is only approved for the seller of an item or an admin, and the state of the auction must be PENDING (the states of an Auction are PENDING, ACTIVE, OVER, CANCELLED, FINALIZED). `stopAuction()` is only approved for an admin, and the state of the auction must be either PENDING or ACTIVE. Both actions are POST requests since that create a Cancellation object associated (and possessed by) an existing Auction object.

The auctions-service publishes a handful of messages to RabbitMQ. They are a mixture of fanout and direct:
* `auction.end` (contains all finalized auction details)
    * example of reactivity: `cam-service` consumes the data and saves it. Henceforth people can access this service to e.g. analyze this data. `shopping-cart-service` uses this information to execute a `fullCheckout()`, which furthermore contacts `user-service` and `item-service` for involved user and item details. It then sends a request (synchronously through a RESTful API) to mark the items bought at the conclusion. In reality, the better design would be to have shopping cart publish an `item.bought` event, which item-service consumes to mark items bought. `item-service` only cares about whether an item is bought to support queries, so this exchange of information need not be instantaneous.
* `auction.new-high-bid`
    * example of reactivity: `notification-service` emails / notifies the associated bidder(s) involved and the seller 
* `auction.start-soon`
    * example of reactivity: `notification-service` sees the item it needs to send out an alert for. It asks the item-service for all the user_id's who have bookmarked an item. The `notification-service` knows all their emails, and it sends an alert to all of them.
* `auction.end-soon`

![alt text](misc/auction-interface.jpg "auctions-service")

> illustration of `auction-service`'s interface. solid lines represent synchronous communication through HTTP requests (to an exposed RESTful API). Red dotted lines indicate asynchronous communication. Not mentioned so far: `shopping-cart-service`'s checkout() operation includes, for each item, a synchronous call to cancel the auction associated with that item (N calls for N items). ShoppingCart already knows the start time of each auction, so it knows before making the calls to the auctions-service that the cancellations can be done (we opted for only being able to "buy now" items if the current time is prior to auction start time). Thus if cancellations fail for any reason, the shopping-cart can still proceed with the transaction. Upon failure, the auction concludes, and if there is a winner, he will not be allowed to buy the item because shopping-cart will see the item has already been bought. Thus, worst case scenario the failure of these calls results in a handful of people wasting their time bidding in an action when the item has already been sold.

auctions-service publishes new bids to an exchange (fanout) rather than directly to itself is to support a possible future refactor to support a `real-time-views` service and create a performant system. The idea: the auctions-service will have heavy traffic at the end of an auction to create new bids and to get all the existing bids (queries). By creating a dedicated service for queries, we can approximately half the traffic to the `auction-service` so that it can focus on bid processing, giving a more real-time experience.

![alt text](misc/auction-refactor.jpg "auctions-refactor")
> the `real-time-views` service consumes asynchronous messages about incoming bids and about the result of processing bids. It can maintain essentially a sorted table of bids that users can see, and the table can be updated to reflect the state of the bid (received and being processed, approved and official, etc). This allows bidders to react to bids with very low latency.

# testing

some microservices have unit tests for e.g. domain layer or database layer type code. Few tests exist for integration. Integration was done through manual inspection of correctness with sequences of requests defined in a large Postman workspace we created. It has workflows of typical system requests in the backend.