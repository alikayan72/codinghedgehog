+++
date = '2025-09-07T14:14:00-04:00'
draft = true
title = 'A Synthetic Stock Price Generator in GoLang: Part 2'
+++ 

# A Synthetic Stock Price Generator in Go: Part 2

## Intro

In part 1, we created a service that can publish made up live stock prices. But what if we want to have some data backfilled already from a given start time? In this post, we will extend our service so that it can generate data from a given date/time and flow into live data. We will also have a chance to see Go’s “time” library in action

All the code in this article can be found on my [GitHub](https://github.com/alikayan72/hedgy). 

You can find the full blog series [here](/)

## Updating the Proto

We need to let our users pass a start date time. Let’s update our proto

```protobuf
message StartMarketRequest {
  repeated string stocks = 1;
  google.protobuf.Timestamp start_date = 2;
}
```

Don’t forget to run the proto compiler after the change. You can see how this can be achieved [here](/posts/a-synthetic-stock-price-generator-in-go-part-1/#generating-the-go-code-for-the-protobuf)

## Start Date For the Stock Struct

When we make a our structs they should know about the start date so that they can set their timestamps. let’s make that quick update

```go
func MakeStock(id string, timeStamp time.Time) *Stock {
	data := stockpb.Stock{
		Id: id, TimeStamp: timestamppb.New(timeStamp),
		Last: 0, Volume: 0, TotalVolume: 0, Volatility: 0.000082,
	}
	stock := Stock{Data: &data, gen: MakeGenerator()}
	stock.Data.Last = stock.gen.GetInitialPrice()
	return &stock
}

```

Notice that internally we work with time.Time structs. However, the stockpb time stamp is a protobuf.timestamp so we need to make the conversion when we are passing this to the time stamp

## Making the Market Work with Start Time

The market also needs to know about the start time. It should generate all the stocks with the given start time, and it should also know what the current time is. So let’s update the struct so that it has a concept of “market time”. We will also update the MakeMarket function

```go
type Market struct {
	Stocks    []*g.Stock
	marketTime time.Time
}

func MakeMarket(stockIds []string, startTime time.Time) *Market {
	stocks := make([]*g.Stock, 0)
	for _, id := range stockIds {
		stock := g.MakeStock(id, startTime)
		stocks = append(stocks, stock)
	}

		return &Market{stocks, startTime}
}
```

### Generating Past Points

Before we start the live data we need to generate past points for each of the symbols in the market. This will be the same as how we generate the live data, except we don’t need to throttle the loop every second. We will just increment the marketTime by 1 second each iteration. After this we flow into our live loop as before

```go
func (m *Market) Run(ctx context.Context) <-chan g.Stock {
	ch := make(chan g.Stock)

	go func() {
		defer close(ch)

		// generate data from startTime to Now
		clockTime := time.Now()
		// We generate new points as long as market time < clock time which is when the request is made. We advance by 1 second every iteration
		for ; m.marketTime.Compare(clockTime) == -1; m.marketTime = m.marketTime.Add(time.Second * 1) {
			for _, stock := range m.Stocks {
				stock.Advance(m.marketTime)
				ch <- *stock
			}
		}

		// generated past points keep publishing live points
		for {
			m.marketTime = time.Now()
			select {
			case <-ctx.Done():
				return
			default:
				for _, stock := range m.Stocks {
					stock.Advance(m.marketTime)
					ch <- *stock
				}
				time.Sleep(1 * time.Second)
			}
		}
	}()

	return ch
}
```

I ran into a few gotchas with go’s “time” library here. I professionally use C# at work which has more syntactic sugar than go, but as always I see go’s explicitness a plus

1. Compare function is quite useful. a.Compare(b) returns -1 if a < b, +1 if a > b, and 0 if a == b. 
    1. Turns out I missed something fundamental about go that it doesn’t have operator overloading. My first inclination to compare the time structs was to use the “<” operator, but then I learned that there is no overloading for operators. 
2. You can use time.Add to add some duration to a time struct. You can use time.Second, time.Millisecond, time.Hour etc
    1. Again no operator overloading so has to use the Add function. 
    2. Another thing is that .Add returns a new struct, it doesn’t update the original one in place, which is why we do the assignment m.marketTime = m.marketTime.Add(time.Second * 1)

## Updating the StartMarket RPC

Lastly, we need to pass the start time from the request to the market. Before we do that we check if the StartDate exists in the request. If not, our start date is now. 

```go
func (s *stockServer) StartMarket(req *stockpb.StartMarketRequest, stream grpc.ServerStreamingServer[stockpb.Stock]) error {
	startTime := time.Now()

	if req.StartDate != nil {
		fmt.Println("start date passed")
		startTime = req.StartDate.AsTime()
	}
	
	market := MakeMarket(req.Stocks, startTime)
	stockChannel := market.Run(stream.Context())
	for update := range stockChannel {
		// data changes when i write to stream. either copy everything or come up with better solution
		if err := stream.Send(update.Data); err != nil {
			return err
		}
	}
	return nil
}
```

Notice that if proto.Timestamp is not passed it will be nil. If it’s passed we can convert that value to a time struct with .AsTime()

Now, make some requests with and without the start time. There is a significant bug with historic data generation where it’s sending the same point multiple times, and jumping in time. We also have a minor bug without the start time where the initial point is sent twice

## Next up

We will tackle the bugs and talk about- *spoiler alert*- pointers because they just ruined our historic data
