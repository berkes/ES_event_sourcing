---
author:
  - Bèr `berkes` Kessels
title: Event Sourcing
subtitle: "Event Sourcing basics - Nimma.codes 25-10-2022"
abstract: "Event sourcing is een software-architectuur waarbij software haar interne toestand niet direct bijhoudt in een database, maar door het lezen en vastleggen van gebeurtenissen in een event store. Zoals de balans op je bankrekening niet een cijfertje in een database is, maar een afgeleide van een hele serie transacties op die bankrekening. We lopen door wat voorbeelden heen, leren wanneer event-sourcing een goede match is, en welke architectuur-patronen er vaak omheen worden ingezet"
keywords: es, eventsourcing, ddd
lang: nl
---

# Event Sourcing {background-image="./dvd_screensaver.gif"}

## Over deze presentatie

* Is online: [berk.es/es](https://berk.es/es) (github.com/berkes/es)
* Bevat alle links
* Is in het Nederlands

## Over mij {background-image="./ber.jpg"}

* Bèr `berkes` Kessels
* @berkes - LinkedIn, Twitter, Fediverse
* CTO at [REDACTED]
* Blog berk.es

## Doel

* Wegwijs worden in de chaos van TLAs.
* Een volgend project met ES opzetten?

# Event Sourcing is simpel.

---

![Bank app](./bank_app.png)

---

Aggregate, Left Fold, Reduce, Projection, etc.

---

We slaan alle transacties op - events. Balans krijgen we door ze te doorlopen.

---

* Wat was onze balans op vrijdag de 13e om 13:37?
* Hoeveel dagen hebben we rood gestaan?
* Teken een diagram van ons loon over de jaren heen.

---

![Cart Flow](./cart_flow_event_sourced.png)

---

:::::::::::::: {.columns align=top .onlytextwidth}
::: {.column width="60%" align=center} 

* Wat zouden we een klant die een balpen koopt nog meer aanraden?
* Bij welke prijzen gaan klanten producten uit de winkelwagen halen?

:::
::: {.column width="40%"}
![Cart Flow](./cart_flow_event_sourced.png)
:::
::::::::::::::

---

* Audit Log
* Toekomstige analyse
* Data representeert de werkelijkheid

---

![Journal Bookkeeping](./bookkeeping.jpg)

---

* Git
* Bank - Boekhouden
* Blockchain
* AWS

---

> Het algemene doel van een logboek is dat later teruggelezen kan worden wat er precies gebeurd is op een bepaald moment. Op het moment van opschrijven is dus nog niet bekend wat men later terug wil lezen. Het is dus van groot belang dat alle details in het logboek terechtkomen, zodat men later niets mist.

---

> Ook is het van belang dat de vastleggingen in een logboek authentiek zijn. De gegevens mogen niet verwijderd of ongecontroleerd gemuteerd worden. 

[Wikipedia Logboek](https://nl.wikipedia.org/wiki/Logboek)

---

![Events: Immutable. Event Log: Append Only](./immutable.jpg)

---

* Hoe herstel je een foute "X aangemaakt" of "Y verwijderd"?
* Met een tegengestelde "X weggehaald" of "Y toegevoegd".
* En nooit door X-aangemaakt of Y-verwijderd te verwijderen.

---

![DDD Book cover](./dddbook.jpg)

---

```ruby
##
# A position is the amount of a security, asset, or property 
# that is owned (or sold short) by some individual or other entity.
class Position
  attr_reader :total_buying_price, :amount

  def initialize(ticker:, currency:)
    @events = []
    @ticker = ticker
    @currency = currency
    @total_buying_price = 0
    @amount = 0
  end

  def buy(amount:, price:)
    handle_event(
      PositionIncreased.new(
        amount: amount,
        price: price
      )
    )
  end

  def sell(amount:, price:)
    fail "Cannot sell more than #{@amount}" if amount > @amount
    handle_event(PositionReduced.new(amount: amount, price: price))
  end

  private

  attr_reader :events

  ##
  # Handles an incoming event.
  # For now, we keep it simple: no DSLs or fancy metaprogramming, just a switch.
  def handle_event(event)
    @events << event
    case event
    when PositionIncreased then handle_position_increased(event)
    when PositionReduced   then handle_position_reduced(event)
    end
  end

  ##
  # Our handle_event calls this method for each PositionAdded event
  def handle_position_increased(event)
    @amount += event.amount
    @total_buying_price += event.amount * event.price
  end

  ##
  # Our handle_event calls this method for each PositionAdded event
  def handle_position_reduced(event)
    @amount -= event.amount
  end
end

class PositionIncreased < OpenStruct
end
class PositionReduced < OpenStruct
end
```

::: notes

- Domain models with Domain meaning and domain specific API.
- Events are past tense.
- Validation done before storing event.
- It doesn't yet store events.

:::

---

```ruby
aapl = Position.new(ticker: 'AAPL', currency: 'USD')
aapl.buy(amount: 2, price: 131.13)
aapl.sell(amount: 1, price: 142.00)
aapl.buy(amount: 3, price: 172.42)
puts aapl.total_buying_price
# 779.52
puts aapl.amount
# 4
```

---

* Heel makkelijk te testen.
* Zeer eenvoudig uit te breiden.
* Business logic: cohesive.

---

```ruby

class Aggregate
  def load(id, event_repo)
    instance = new(id, event_repo)
    events = event_repo.get_all_for(id)
    events.each do |event|
      instance.handle_event(event)
    end
    instance
  end

  def initialize(id, event_repo)
    @id = id
    @event_repo = event_repo
    @version = 0
  end

  def handle_event(event)
    @version++
    event_repo.add(event)
  end
end

class Position < Aggregate
  def initialize(id, event_repo)
    super(id, event_repo)
    @opened = false
  end

  def open(ticker: ticker, currency: currency)
    handle_event(PositionOpened.new(ticker: ticker, currency: currency))
  end

  def buy()
    fail "must first open the position before buying stocks" unless @opened
    #...
  end

  def handle_event(event)
    super(event)
    # ...
  end

  def handle_open(event)
    @ticker = event.ticker
    @currency = event.currency
    @opened = true
  end
end

aapl = Position.load('AAPL-USD', pg_event_store)
aapl.open(ticker: 'AAPL', currency: 'USD')
aapl.buy(amount: 2, price: 131.13)
aapl.sell(amount: 1, price: 142.00)

aapl = Position.load('AAPL-USD', pg_event_store)
aapl.buy(amount: 3, price: 172.42)
puts aapl.total_buying_price
# 779.52
puts aapl.amount
# 4

```

---

## The world of ES

![The world of ES](./ddd.png)

## Pro's and cons

## ES everything

## Resources

* Domain-Driven Design: Tackling Complexity in the Heart of Software - Eric Evans
* Implementing Domain-Driven Design - Vaugh Vernon
* https://martinfowler.com/eaaDev/EventSourcing.html - Martin Fowler
* https://www.youtube.com/watch?v=LDW0QWie21s - DDD, CQRS and Event Sourcing - Greg Young
