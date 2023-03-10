---
title: "How to Unmarshal JSON in a custom way in Golang"
datePublished: Fri Mar 10 2023 03:58:18 GMT+0000 (Coordinated Universal Time)
cuid: clf20cjcx000e0aju3htdeptg
slug: golang-custom-json-unmarshal
tags: go

---

When I was working on [gocoinex library](https://github.com/arshamalh/gocoinex) for the first time, I needed some custom configuration on JSON unmarshalling process, I want to share my learnings with you here, but before that, we're going to have a recap on how we can unmarshal JSON in golang.

> These are real world examples. (coinex exchange API)

First, we make our request like this:

```go
raw_response, _ := http.Get("https://api.coinex.com/v1/market/list")
```

And we'll get a JSON object like this:

```json
{
    "code": 0,
    "data": [
        "LTCBCH",
        "ETHBCH",
        "ZECBCH",
        "DASHBCH"
    ],
    "message": "Ok"
}
```

But we should parse it, There are different ways to parse it, You can use **NewDecode** or **Unmarshal** functions of the **JSON** package:

* Use **NewDecoder.Decode** when dealing with `io.Reader`**.**
    
* Use **Unmarshal** when you have a `[]byte.`
    

And you can decode it to a `struct` or to a `map[string]interface{}` :

* Decode to `map[string]interface{}` when you don't know what the message should look like.
    
* Decode to your custom named `struct` in other cases.
    

In this case, I prefer NewDecoder and struct combination, but it's still OK to follow other combinations.

So we should make a struct like this:

```go
type AllMarketList struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    []string
}
```

Also, we can have embedded structs, for example, we break the last struct into two:

```go
type GeneralResponse struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

type AllMarketList struct {
    GeneralResponse
    Data    []string
}
```

And there is no difference.

Finally, use NewDecoder to decode the raw\_response to the AllMarketList struct:

```go
var allMarketList AllMarketList
json.NewDecoder(raw_response.Body).Decode(&allMarketList)
```

## Completed code

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type AllMarketList struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    []string
}

func main() {
    raw_response, _ := http.Get("https://api.coinex.com/v1/market/list")

    var allMarketList AllMarketList
    if err := json.NewDecoder(raw_response.Body).Decode(&allMarketList); err != nil {
        fmt.Println(err)
    }
    defer raw_response.Body.Close()
    fmt.Printf("%+v\n", allMarketList)
}
```

## Example Two

Think we have a JSON like this:

```json
{
  "code": 0,
  "data": {
    "date": 1513865441609, # server time when returning
    "ticker": {
        "open": "10", # highest price
        "last": "10.00", # latest price 
        "vol": "110" # 24H volume
    }
  },
  "message" : "Ok"
}
```

We're going to improve some things in the decoding process

1. In this case, the Unix timestamp can't be parsed, we should provide a solution.
    
2. We want to remove the "ticker" key and access "open", "last", and "vol" directly from "data".
    
3. "last" should be exported to the field named "Close"! (same happens with "vol" and "Volume".
    
4. "open", "last", "vol" should be float, not string, but we'll leave them for the following example.
    

Problems 1 and 2, can be solved by implementing the UnmarshalJSON method on any struct we want to decode. problem 3 will be easily solved with JSON tags. (I've mentioned it in the code below)

Our final struct should be like this:

```go
// Final struct
type SingleMarketStatistics struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    TickerData
}

// Inner struct that we should implement to solve problem 2
type TickerData struct {
    ServerTime CTime   `json:"date"` // CTime is short for CustomTime
    Open       float64 `json:"open"`
    Close      float64 `json:"last"` // Different Attribute name and tag name
    Volume     float64 `json:"vol"` // Different Attribute name and tag name
}

// Custome time
// Inner struct that we should implement to solve problem 1
type CTime struct {
	time.Time
}
```

## Custom time implementation

```go
func (t *CTime) UnmarshalJSON(data []byte) error {
    // Ignore null, like in the main JSON package.
    if string(data) == "null" || string(data) == `""` {
        return nil
    }
    // Fractional seconds are handled implicitly by Parse.
    i, err := strconv.ParseInt(string(data), 10, 64)
    update := time.UnixMilli(i)
    *t = CTime{update}
    return err
}
```

And we don't get an error anymore! This method will be automatically used (thanks to interfaces!) whenever we want to decode a time to CTime!

## Custom Data implementation

```go
func (t *TickerData) UnmarshalJSON(data []byte) error {
    if string(data) == "null" || string(data) == `""` {
        return nil
    }

    // This is how this json really looks like.
    var realTicker struct {
        ServerTime CTime `json:"date"`
        Ticker     struct {
            // tags also can be omitted when we're using UnmarshalJSON.
            Open   string `json:"open"`
            Close  string `json:"last"`
            Volume string `json:"vol"`
        } `json:"ticker"`
    }

    // Unmarshal the json into the realTicker struct.
    if err := json.Unmarshal(data, &realTicker); err != nil {
        return err
    }

    // Set the fields to the new struct,
    // with any shape it has,
    // or anyhow you want.
    *t = TickerData{
        ServerTime: realTicker.ServerTime,
        Open:       realTicker.Ticker.Open,
        Close:      realTicker.Ticker.Close,
        Volume:     realTicker.Ticker.Volume,
    }

    return nil
}
```

Now, use NewDecoder as before, no change is needed.

```go
var singleMarketStatistics SingleMarketStatistics 
json.NewDecoder(raw_response.Body).Decode(&allMarketList)
```

## Example Three

Imagine a JSON like this:

```json
{
  "asks": [ // This is a array of asks
    [ // This is a array of ONE ask
      "10.00", // Price of ONE ask
      "0.999", // Amount of ONE ask
    ]
  ],
  "bids": [ // Same structure as asks
    [
      "10.00",
      "1.000",
    ]
  ]
}
```

As it's totally clear, in a non-professional way, we should decode "asks" to `[][]string` and access first Ask price like this `asks[0][0]` and amount `asks[0][1]` .  
Seriously, Who remembers 0 is the price and 1 is the amount? What was which? 😄  
So we'll manage them on the UnmarshalJSON method.  
Also, we'll solve problem 4 of the previous example that also exists here.

```go
type BidAsk struct {
    // Tags are not needed.
    Price  float64 `json:"price"`  // Bid or Ask price
    Amount float64 `json:"amount"` // Bid or Ask amount
}

func (t *BidAsk) UnmarshalJSON(data []byte) error {
    // Ignore null, like in the main JSON package.
    if string(data) == "null" || string(data) == `""` {
        return nil
    }

    // Unmarshal to real type.
    var bisask []string
    if err := json.Unmarshal(data, &bisask); err != nil {
        return err
    }

    // Change value type from string to float64.
    price, err := strconv.ParseFloat(bisask[0], 64)
    if err != nil {
        return err
    }
    amount, err := strconv.ParseFloat(bisask[1], 64)
    if err != nil {
        return err
    }

    // Map old structure to new structure.
    *t = BidAsk{
        Price:  price,
        Amount: amount,
    }
    return err
}

type MarketDepth struct {
    Asks   []BidAsk `json:"asks"` // Ask depth
    Bids   []BidAsk `json:"bids"` // Bid depth
}
```

Again, we simply use the following:

```go
var marketDepth MarketDepth 
json.NewDecoder(raw_response.Body).Decode(&marketDepth)
```

And enjoy the beauty of your result:

```go
for i, ask := range data.Data.Asks {
    fmt.Printf("Ask %v\n", i)
    fmt.Printf("  Price: %v\n", ask.Price) // here is the beauty
    fmt.Printf("  Amount: %v\n", ask.Amount) // here is the beauty
    fmt.Println()
}
for i, bid := range data.Data.Bids {
    fmt.Printf("Bid %v\n", i)
    fmt.Printf("  Price: %v\n", bid.Price) // here is the beauty
    fmt.Printf("  Amount: %v\n", bid.Amount) // here is the beauty
    fmt.Println()
}
```