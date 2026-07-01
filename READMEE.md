# trybit-go

Go client for the [Trybit](https://trybit.com) crypto-processing API.

![Go](https://img.shields.io/badge/Go-1.21+-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)
[![Go Reference](https://pkg.go.dev/badge/github.com/trybit-go/trybit.svg)](https://pkg.go.dev/github.com/trybit-go/trybit)
[![Go Report Card](https://goreportcard.com/badge/github.com/trybit-go/trybit)](https://goreportcard.com/report/github.com/trybit-go/trybit)

No external dependencies — standard library only.

```
go get github.com/trybit-go/trybit
```

## Example

```go
package main

import (
	"context"
	"fmt"
	"log"

	trybit "github.com/trybit-go/trybit"
)

func main() {
	c, err := trybit.New("YOUR_PROJECT_API_KEY")
	if err != nil {
		log.Fatal(err)
	}

	inv, err := c.Invoice.Create(context.Background(), trybit.InvoiceCreateParams{
		ShopID:   "xBAivfPIbskwuEWj",
		Amount:   100,
		Currency: trybit.FiatUSD,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Pay at:", inv.Link)
}
```

## Coverage

Invoices and account endpoints so far:

| Method | Endpoint |
|---|---|
| `Invoice.Create` | `POST /invoice/create` |
| `Invoice.Cancel` | `POST /invoice/merchant/canceled` |
| `Invoice.List` | `POST /invoice/merchant/list` |
| `Invoice.Info` | `POST /invoice/merchant/info` |
| `Invoice.Statistics` | `POST /invoice/merchant/statistics` |
| `Balance.All` | `POST /merchant/wallet/balance/all` |

Payouts, postback webhooks and static wallets aren't wrapped yet — they're next.

## Usage

```go
// Optional fields go straight on the params struct; the client packs the
// ones that belong in add_fields for you.
inv, _ := c.Invoice.Create(ctx, trybit.InvoiceCreateParams{
	ShopID:         "shop",
	Amount:         100,
	Currency:       trybit.FiatEUR,
	Cryptocurrency: trybit.CryptoUSDTTRC20,
	TimeToPay:      &trybit.TimeToPay{Hours: 24},
})

// Fetch up to 100 invoices by UUID
invs, _ := c.Invoice.Info(ctx, "INV-89UX09KA", "INV-OTHER")

// List a date range
page, _ := c.Invoice.List(ctx, trybit.InvoiceListParams{
	Start: trybit.NewDate(2026, 1, 1),
	End:   trybit.NewDate(2026, 1, 31),
	Limit: 50,
})
fmt.Println(page.AllCount, len(page.Invoices))

// Cancel, get stats, check balances
_ = c.Invoice.Cancel(ctx, "INV-89UX09KA")
st, _ := c.Invoice.Statistics(ctx, trybit.NewDate(2026, 1, 1), trybit.NewDate(2026, 1, 31))
balances, _ := c.Balance.All(ctx)
```

Errors from the API come back as `*trybit.APIError`, so you can pull out the
status code and the raw fields:

```go
var apiErr *trybit.APIError
if errors.As(err, &apiErr) {
	fmt.Println(apiErr.StatusCode, apiErr.Fields)
}
```

## Amounts

All the amount fields are `float64`. That's fine for showing values, but
`float64` can't hold every crypto amount exactly (some go to 18 decimals) and
you get the usual floating-point rounding. So don't add up balances or compare
amounts for equality straight off these fields — convert to a decimal type
first if it matters.
