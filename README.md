# API

## Serwer testowy
API produkcyjne dostępne jest pod adresem https://api.inpay.pl
Natomiast adres serwera testowego to: api.test-inpay.pl
W ramach testowania usługi prosimy o posługiwanie się portfelami typu Testnet np.: Bitcoin Wallet Testnet.

W celu zasilenia Portfela Testnet można wykorzystać serwisy:
faucet.xeno-genesis.com/
http://www.royalforkblog.com/2014/12/09/testnet-faucet/

Przykładowa implementacja klienta PHP dostępna w repozytorium Github https://github.com/inpay/phplib-inpay
Integracja z rozwiązaniami e-commerce wymaga jedynie wykorzystaniu metody /invoice/create. Służy ona do wygenerowania linku płatności na który należy przekierować płatnika. Dobrą praktyką jest również przekazanie jako parametr adresów, na które InPay będzie mógł wysłać powiadomienia (callback) z informacjami i statusach danej transakcji. Rozwiązanie z wykorzystaniem adresów powiadomień jest rozwiązaniem pasywnym. Alternatywnie status każdej płatności może sprawdzać za pomocą metody `/invoice/status`


### Uwierzytelnianie
W systemie InPay musi być założone aktywne konto. Z systemu InPay lub od administratorów należy pobrać:

Attribute | Meaning
------------ | -------------
**apiKey** | Your public API key
**secretApiKey** | Your secret API key used for message signing and verification. Do not share with anyone


### /invoice/create
Metoda /invoice/create (http://api.test-inpay.pl/invoice/create). przyjmuje następujące argumenty POST (form-data):

Attribute | Meaning
------------ | -------------
**apiKey** | string klucz API
**amount** | float Kwota do zapłaty 123.25
**currency** | string currency, aktualnie obsługiwane PLN, EUR, USD
**optData** | string opcjonalny urlencode data key=value&key2=value2, zostaną dodane do powiadomienia o statusie transakcji **orderCode** | string opcjonalny identyfikator zamówienia
**description** | string opcjonalny opis zamówienia
**customerEmail** | string opcjonalny email
**callbackUrl** | url opcjonalny URL na który mają być wysyłane powiadomienia callback
successUrl** | url opcjonalny URL na który płatnik będzie kierowaniu po udanej transakcji
**failUrl** | url opcjonalny URL na który zostanie przekierowany użytkownik jeśli płatność się nie uda
**minConfirmations** | int opcjonalny minimalna liczba potwierdzeń sieci Bitcoin, wymaganych do zmiany statusu na confirmed. domyślna i minimalna wartość: 6

Wartość zwracana:

Attribute | Meaning
------------ | -------------
**invoiceCode** | identyfikator faktury nadawany przez InPay
**redirectUrl** | URL na który należy przekierować płatnika

### Inicjowanie płatności
Należy zainicjować przekierowanie na `redirectUrl`.

Płatność kończy się przekierowaniem na succcessUrl `failUrl`. Adres `callbackUrl` (POST) :

Attribute | Meaning
------------ | -------------
**orderCode** | identyfikator zamówienia
**amount** | potwierdzona kwota do wypłaty na konto pomniejszona o prowizje
**currency** | waluta wypłaty na konto
**fee** | wysokość pobranej prowizji
**invoiceCode** | identyfikator faktury w systemie
**status** | status faktury
**optData** | opcjonalne dane podane przy funkcji `/invoice/create`

callbackUrl | posiada w nagłówku atrybut API-Hash. Jest to skrót SHA512 wygenerowany ze zmiennych wysyłanych za pomocą POST & sekretny klucz API.

Sprawdzenie wartości powyższej zmiennej jest istotne, aby uniknąć ew. zagrożeń typu man-in-the-middle.
Przykładowy kod sprawdzający `API-Hash`:

```php
  function checkApiHash() {
    $secretApiKey = 'yourPrivateAPIkeySecret';
    $apiHash = $_SERVER['HTTP_API_HASH'];
    $query = http_build_query($_POST);
    $hash = hash_hmac("sha512", $query, $secretApiKey);
    return $apiHash == $hash;
  }
```

### /invoice/status
Opcjonalnie w przypadku rozwiązań dedykowanych np. płatności na urządzeniach mobilnych lub innych nietypowych aplikacjach do wykorzystania jest również metoda `/invoice/status`, przyjmuje argument POST (form-data) invoiceCode. Wartość zwracana: (JSON object).

Attribute | Meaning
------------ | -------------
**open_date** | data utworzenia (2014-04-15 16:01:01)
**close_date** | data zapłaty (2014-04-15 16:01:01)
**expire_date** | data wygaśnięcia płatności (2014-04-15 16:01:01)
**secondsFromOpenToExpire** | czas ważności płatności w sekundach (1200)
**secondsToExpire** | pozostały czas ważności w sekundach (1142)
**received_amount** | otrzymana kwota (1234567 Satoshi = 0.00000001 BTC)
**received_currency** | kryptowaluta
**in_amount** | oczekiwana kwota do wypłaty partnerowi * 100 (np. 1000 dla 10 PLN)
**in_currency** | oczekiwana waluta kwoty wpłacanej partnerowi PLN, EUR, USD
**expected_amount** | oczekiwana kwota w kryptowalucie(1234567)
**expected_currency** | oczekiwana waluta wpłacanej kwoty satoshi
**input_address** | adres w sieci Bitcoin aktualnej aktywnej płatności (mwTNRX6A1E9TNH9JcLqt9eVhJo6zeDsYs7)
**btcAmount** | kwota płatności wyrażona w BTC (0.01192708)
**btcLink** | adres url zgodny z protokołem BIP 0021 (bitcoin:mwTNRX6A1E9TNH9JcLqt9eVhJo6zeDsYs7?amount=0.01192708)
**invoice_code** | Identyfikator faktury aktulanej płatności: (PURGAM4BQHN9J3L)
**status** | status płatności
**confirmations** | liczba potwierdzeń z sieci bitcoin

Aby pobrać aktualne źródło obrazu image/png kodu QR umożliwiający zapłatę należy wywołać url `/qr/invoice/{invoice_code}`
gdzie `{invoice_code}` to identyfikator faktury. Należy pamiętać, że kod QR należy przeładować po wygasniętu aktualnej płatności przypisanej do faktury.

### Statusy płatności

Attribute | Meaning
------------ | -------------
**new** | nowa płatność, oczekujemy na wpłatę
**received** | płatność Bitcoin została odnotowana
**confirmed** | otrzymano poprawną wpłatę, transakcja została zatwierdzona przez InPay
**expired** | płatność wygasła
**invalid** | zapłacono za małą kwotę
**overpaid** | zapłacono za dużą kwotę
**refund** | zwrócono środki płacącemu

Funkcja zwróci następujące zmienne:

Attribute | Meaning
------------ | -------------
**success** | informacja dot. tego czy wywołanie się powiodło, jeśli jest false, to znaczy że nastąpił błąd
**payment_hash** | identyfikator płatności
**open_date** | data utworzenia płatności
**close_date** | data zamknięcia płatności
**expire_date** | data wygaśnięcia płatności w przypadku braku wpłaty
**received_amount** | otrzymane środki
**received_currency** | otrzymana waluta
**in_amount** | żądana kwota (przy czym jest to kwota * 100, czyli dla PLN w groszach)
**in_currency** | żądana waluta
**expected_amount** | oczekiwana kwota zapłaty w satoshi (najmniejsza podzielna część Bitcoin)
**expected_currency** | oczekiwana waluta, obecnie zawsze satoshi

Płatność może przyjąć następujący status:

Attribute | Meaning
------------ | -------------
**new** | nowa płatność, oczekujemy na wpłatę
**received** | płatność Bitcoin została odnotowana
**confirmed** | otrzymano poprawną wpłatę, transakcja została zatwierdzona przez InPay
**paid** | dana transakcja została rozliczona, środki wypłacone na rachunek bankowy partnera
**expired** | płatność wygasła
**invalid** | zapłacono za małą kwotę
**overpaid** | zapłacono za dużą kwotę
**refund** | zwrócono środki płacącemu

Status należy sprawdzać do czasu uzyskania statusu `confirmed`
