# trybit-go

**A Go client for the [Trybit](https://trybit.com) crypto-processing API** — invoices, balances, and statistics.

![Go](https://img.shields.io/badge/Go-1.21+-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)
[![Go Reference](https://pkg.go.dev/badge/github.com/trybit-go/trybit.svg)](https://pkg.go.dev/github.com/trybit-go/trybit)
[![Go Report Card](https://goreportcard.com/badge/github.com/trybit-go/trybit)](https://goreportcard.com/report/github.com/trybit-go/trybit)

Covers the Trybit invoice and account API. No external dependencies — standard library only.

## Install

```
go get github.com/trybit-go/trybit
```

Requires Go 1.21 or later.

## Quick start

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

## API coverage

Payouts, postback webhooks, and static wallets are planned for a later release.

| Method | Endpoint | Description |
|---|---|---|
| `Invoice.Create` | `POST /invoice/create` | Create an invoice |
| `Invoice.Cancel` | `POST /invoice/merchant/canceled` | Cancel a `created` invoice |
| `Invoice.List` | `POST /invoice/merchant/list` | List invoices by date range (paginated) |
| `Invoice.Info` | `POST /invoice/merchant/info` | Fetch up to 100 invoices by UUID |
| `Invoice.Statistics` | `POST /invoice/merchant/statistics` | Invoice counts and amounts by status |
| `Balance.All` | `POST /merchant/wallet/balance/all` | Balances across all currencies |

## Usage

### Invoices

```go
// Create with optional fields (placed into add_fields automatically)
inv, _ := c.Invoice.Create(ctx, trybit.InvoiceCreateParams{
	ShopID:         "shop",
	Amount:         100,
	Currency:       trybit.FiatEUR,
	Cryptocurrency: trybit.CryptoUSDTTRC20, // pre-select payment currency
	TimeToPay:      &trybit.TimeToPay{Hours: 24},
})

// Look up specific invoices (up to 100 at once)
invs, _ := c.Invoice.Info(ctx, "INV-89UX09KA", "INV-OTHER")

// List by date range, paginated
page, _ := c.Invoice.List(ctx, trybit.InvoiceListParams{
	Start: trybit.NewDate(2026, 1, 1),
	End:   trybit.NewDate(2026, 1, 31),
	Limit: 50,
})
fmt.Println(page.AllCount, len(page.Invoices))

// Cancel a created invoice
_ = c.Invoice.Cancel(ctx, "INV-89UX09KA")

// Statistics grouped by status
st, _ := c.Invoice.Statistics(ctx, trybit.NewDate(2026, 1, 1), trybit.NewDate(2026, 1, 31))
fmt.Println(st.Count.Paid, st.Amount.Paid)
```

### Balance

```go
balances, _ := c.Balance.All(ctx)
for _, b := range balances {
	fmt.Printf("%s: %g (%g available)\n",
		b.Currency.Code, b.BalanceCrypto, b.AvailableBalance)
}
```

### Errors

Non-2xx responses are returned as `*trybit.APIError`:

```go
inv, err := c.Invoice.Create(ctx, params)
var apiErr *trybit.APIError
if errors.As(err, &apiErr) {
	if apiErr.StatusCode == http.StatusUnauthorized {
		// invalid token
	}
	fmt.Println(apiErr.Fields) // raw error fields from the API
}
```

## A note on monetary precision

Amount fields are `float64`, mirroring the API's JSON numbers. This is fine for reading and displaying values, but `float64` cannot exactly represent every crypto amount (assets go up to 18 decimals) and is subject to binary rounding. Don't reconcile balances or compare amounts for equality directly on these fields — convert to a decimal type first.
