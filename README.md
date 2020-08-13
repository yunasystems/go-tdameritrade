# go-tdameritrade
go client for the tdameritrade api

[![Documentation](https://godoc.org/github.com/JonCooperWorks/go-tdameritrade?status.svg)](https://godoc.org/github.com/JonCooperWorks/go-tdameritrade)


```import "github.com/JonCooperWorks/go-tdameritrade"```

go-tdameritrade handles all interaction with the [TD Ameritrade REST API](https://developer.tdameritrade.com/apis).
See the TD Ameritrade [developer site](https://developer.tdameritrade.com/) to learn how their APIs work.
This is a very thin wrapper and does not perform any validation.
go-tdameritrade doesn't support streaming yet.


## Authentication with TD Ameritrade
There is an example of using OAuth2 to authenticate a user and use the services on the TD Ameritrade API in [examples/webauth/webauth.go](https://github.com/JonCooperWorks/go-tdameritrade/blob/master/examples/webauth/webauth.go).
Authentication is handled by the ```Authenticator``` struct and its methods ```StartOAuth2Flow``` and ```FinishOAuth2Flow```.
You can get an authenticated ```tdameritrade.Client``` from an authenticated request with the ```AuthenticatedClient``` method, and use that to interact with the TD API.
See [auth.go](https://github.com/JonCooperWorks/go-tdameritrade/blob/master/auth.go).

```
// Authenticator is a helper for TD Ameritrade's authentication.
// It authenticates users and validates the state returned from TD Ameritrade to protect users from CSRF attacks.
// It's recommended to use NewAuthenticator instead of creating this struct directly because TD Ameritrade requires Client IDs to be in the form clientid@AMER.OAUTHAP.
// This is not immediately obvious from the documentation.
// See https://developer.tdameritrade.com/content/authentication-faq
type Authenticator struct {
	Store  PersistentStore
	OAuth2 oauth2.Config
}
```

The library handles state generation and the OAuth2 flow.
Users simply implement the ```PersistentStore``` interface (see [auth.go](https://github.com/JonCooperWorks/go-tdameritrade/blob/master/auth.go)) and tell it how to store and retrieve OAuth2 state and an ```oauth2.Token``` with the logged in user's credentials.

```
// PersistentStore is meant to persist data from TD Ameritrade that is needed between requests.
// Implementations must return the same value they set for a user in StoreState in GetState, or the login process will fail.
// It is meant to allow credentials to be stored in cookies, JWTs and anything else you can think of.
type PersistentStore interface {
	StoreToken(token *oauth2.Token, w http.ResponseWriter, req *http.Request) error
	GetToken(req *http.Request) (*oauth2.Token, error)
	StoreState(state string, w http.ResponseWriter, req *http.Request) error
	GetState(*http.Request) (string, error)
}
```

## Interacting with the TD Ameritrade API
The library is centered around the ```tdameritrade.Client```.
It allows access to all services exposed by the TD Ameritrade REST API.
More information about each service can be found on TD Ameritrade's [developer website](https://developer.tdameritrade.com/apis).

```
// A Client manages communication with the TD-Ameritrade API.
type Client struct {
	client *http.Client // HTTP client used to communicate with the API.

	// Base URL for API requests. Defaults to the public TD-Ameritrade API, but can be
	// set to any endpoint. This allows for more manageable testing.
	BaseURL *url.URL

	// services used for talking to different parts of the tdameritrade api
	PriceHistory       *PriceHistoryService
	Account            *AccountsService
	MarketHours        *MarketHoursService
	Quotes             *QuotesService
	Instrument         *InstrumentService
	Chains             *ChainsService
	Mover              *MoverService
	TransactionHistory *TransactionHistoryService
	User               *UserService
	Watchlist          *WatchlistService
}
```

## Examples

More examples are in the [examples](https://github.com/JonCooperWorks/go-tdameritrade/tree/master/examples) directory.

#### Looking up a stock quote using the API.
```
type TDHandlers struct {
	authenticator *tdameritrade.Authenticator
}

func (h *TDHandlers) Quote(w http.ResponseWriter, req *http.Request) {
	ctx := context.Background()
	client, err := h.authenticator.AuthenticatedClient(ctx, req)
	if err != nil {
		w.Write([]byte(err.Error()))
		return
	}

	ticker, ok := req.URL.Query()["ticker"]
	if !ok || len(ticker) == 0 {
		w.Write([]byte("ticker is required"))
		return
	}

	quote, _, err := client.Quotes.GetQuotes(ctx, ticker[0])
	if err != nil {
		w.Write([]byte(err.Error()))
		return
	}

	body, err := json.Marshal(quote)
	if err != nil {
		w.Write([]byte(err.Error()))
		return
	}

	w.Write(body)

}
```

Use at your own risk.
