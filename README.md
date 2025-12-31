# Google Travel からホテルをスクレイピングする方法

[![Promo](https://github.com/bright-jp/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.jp/) 

このガイドでは、Selenium を使う方法、または Bright Data の API を使う方法のいずれかで、Google Travel からホテル一覧、価格、アメニティを収集する方法を説明します。

- [前提条件](#prerequisites)
- [Google Travel から抽出する内容](#what-to-extract-from-google-travel)
- [Selenium でデータを抽出する](#extracting-the-data-with-selenium)
- [Bright Data の Travel API でデータを抽出する](#extracting-the-data-with-bright-datas-travel-api)
    - [Requests](#requests)
    - [AIOHTTP](#aiohttp)
- [Bright Data の代替ソリューション](#bright-datas-alternative-solutions)

## Prerequisites

旅行データをスクレイピングするには、Python と、Selenium／Requests／AIOHTTP モジュールのいずれかが必要です。Selenium を使う場合は、Google Travel から直接ホテル情報をスクレイピングします。Requests と AIOHTTP を使う場合は、Bright Data の [Booking.com API](https://brightdata.jp/products/web-scraper/booking) を使用します。

Selenium を使用する場合は、[webdriver](https://googlechromelabs.github.io/chrome-for-testing/) がインストールされていることを確認してください。Selenium に不慣れな場合は、[こちらのガイド](https://brightdata.jp/blog/how-tos/using-selenium-for-web-scraping) を確認すると、すぐに理解できます。

Selenium をインストールします:

```
pip install selenium
```

Requests をインストールします:

```
pip install requests
```

AIOHTTP をインストールします:

```bash
pip install aiohttp
```

## What To Extract From Google Travel

ホテル結果はすべて、Google Travel のカスタム `c-wiz` 要素内に埋め込まれています。

![Inspect c-wiz Element](https://brightdata.jp/wp-content/uploads/2025/01/image-32.png)

ただし、ページ上には多くの `c-wiz` 要素があります。各ホテルカードには、`div` とこの `c-wiz` 要素の直下にある `a` 要素が含まれています。これらの要素配下のすべての `a` タグを見つけるための CSS セレクタとして、`c-wiz > div > a` を記述できます。

![Inspect a Element](https://brightdata.jp/wp-content/uploads/2025/01/image-33.png)

リスティング名は `h2` に埋め込まれています。

![Inspect h2 Element](https://brightdata.jp/wp-content/uploads/2025/01/image-34.png)

価格は `span` に埋め込まれています。

![Inspect Price Element](https://brightdata.jp/wp-content/uploads/2025/01/image-35.png)

アメニティは `li`（リスト）要素に埋め込まれています。

![Inspect Amenities](https://brightdata.jp/wp-content/uploads/2025/01/image-36.png)

ホテルカードを特定できたら、そこから前述したすべてのデータを抽出できます。

## Extracting The Data With Selenium

Selenium でのデータ抽出は、どこを見るべきかが分かれば比較的簡単です。ただし、Google Travel は結果を動的に読み込むため、事前設定した待機、マウスクリック、カスタムウィンドウなどで成り立つ繊細なプロセスになります。

以下が Python スクリプト全文です:

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.action_chains import ActionChains
import json
from time import sleep

OPTIONS = webdriver.ChromeOptions()
OPTIONS.add_argument("--headless")
OPTIONS.add_argument("--window-size=1920,1080")



def scrape_hotels(location, pages=5):
    driver = webdriver.Chrome(options=OPTIONS)
    actions = ActionChains(driver)
    url = f"https://www.google.com/travel/search?q={location}"
    driver.get(url)
    done = False

    found_hotels = []
    page = 1
    result_number = 1
    while page <= pages:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        sleep(5)
        hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")
        print(f"-----------------PAGE {page}------------------")
        print("FOUND ITEMS: ", len(hotel_links))
        for hotel_link in hotel_links:
            hotel_card = hotel_link.find_element(By.XPATH, "..")
            try:
                info = {}
                info["url"] = hotel_link.get_attribute("href")
                info["rating"] = 0.0
                info["price"] = "n/a"
                info["name"] = hotel_card.find_element(By.CSS_SELECTOR, "h2").text
                price_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span")
                info["amenities"] = []
                amenities_holders = hotel_card.find_elements(By.CSS_SELECTOR, "li")
                for amenity in amenities_holders:
                    info["amenities"].append(amenity.text)
                if "DEAL" in price_holder[0].text or "PRICE" in price_holder[0].text:
                    if price_holder[1].text[0] == "$":
                        info["price"] = price_holder[1].text
                else:
                    info["price"] = price_holder[0].text
                rating_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span[role='img']")
                if rating_holder:
                    info["rating"] = float(rating_holder[0].get_attribute("aria-label").split(" ")[0])
                info["result_number"] = result_number
                
                if info not in found_hotels:
                    found_hotels.append(info)
                result_number+=1
                
            except:
                continue
        print("Scraped Total:", len(found_hotels))
        
        next_button = driver.find_elements(By.XPATH, "//span[text()='Next']")
        if next_button:
            print("next button found!")
            sleep(1)
            actions.move_to_element(next_button[0]).click().perform()
            page+=1
            sleep(5)
        else:
            done = True

    driver.quit()

    with open("scraped-hotels.json", "w") as file:
        json.dump(found_hotels, file, indent=4)

if __name__ == "__main__":
    PAGES = 2
    scrape_hotels("miami", pages=PAGES)
```

スクリプトの処理を手順ごとに確認します:

1. まず `ChromeOptions` のインスタンスを作成します。これを使って `--headless` と `--window-size=1920,1080` 引数を追加します。

> **Note**\
> カスタムウィンドウサイズがない場合、結果が正しく読み込まれず、同じ結果を何度もスクレイピングしてしまいます。

2. ブラウザ起動時に、キーワード引数 `options=OPTIONS` を使用します。これにより、Chrome がカスタムオプション付きで起動します。

3. `ActionChains(driver)` により `ActionChains` インスタンスを取得します。これは後ほど、カーソルを `Next` ボタンへ移動してクリックするために使用します。

4. 実行時間を制御するために `while` ループを使用します。スクレイピングが完了したら、このループを抜けます。

5. `hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")` により、ページ上のすべてのホテルリンクを取得します。親要素は xpath を使って `hotel_card = hotel_link.find_element(By.XPATH, "..")` のように取得します。

6. 先ほど確認した個々のデータをすべて抽出します:
    - url: `hotel_link.get_attribute("href")`
    - name: `hotel_card.find_element(By.CSS_SELECTOR, "h2").text`
    - 価格を探す際、カード内に `DEAL` や `GREAT PRICE` などの追加要素が含まれる場合があります。常に正しい価格を取得できるように、`span` 要素を配列として抽出します。配列にこれらの単語が含まれている場合は、最初の要素（`price_holder[0].text`）ではなく 2 番目の要素（`price_holder[1].text`）を採用します。
    - 評価を探す場合にも `find_elements()` メソッドを使用します。評価が存在しない場合は、デフォルト値として `n/a` を設定します。
    - `hotel_card.find_elements(By.CSS_SELECTOR, "li")` でアメニティの保持要素を取得します。各要素は `text` 属性で抽出します。
7. 必要なページ数をスクレイピングするまでこのループを続けます。データを取得したら、`done` を `True` に設定してループを終了します。
8. ブラウザを閉じ、`json.dump()` を使ってスクレイピングしたデータを JSON ファイルに保存します。

## Extracting the Data With Bright Data’s Travel API

スクレイパーに依存したくない場合や、セレクタやロケータの扱いを避けたい場合は、[travel data](https://brightdata.jp/use-cases/travel) を利用するか、[Booking.com API](https://brightdata.jp/products/web-scraper/booking) を使ってホテルデータを抽出できます。実装方法としては、`requests` モジュールと AIOHTTP ライブラリの 2 つがあります。

### Requests

以下のコードは Booking.com API を使えるように設定します。API key、旅行先、チェックイン日、チェックアウト日を入力するだけです。最初に API へリクエストしてデータ生成をトリガーし、その後 10 秒ごとにレポートの準備ができたかを繰り返し確認します。データを受け取ったら JSON ファイルに保存します。

```python
import requests
import json
import time


def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"

    #booking.com dataset
    dataset_id = "gd_m4bf7a917zfezv9d5"

    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    auth_token = api_key

    #
    headers = {
        "Authorization": f"Bearer {auth_token}",
        "Content-Type": "application/json"
    }

    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    response = requests.post(endpoint, headers=headers, json=payload)

    if response.status_code == 200:
        print("Request successful. Response:")
        print(json.dumps(response.json(), indent=4))
        return response.json()["snapshot_id"]
    else:
        print(f"Error: {response.status_code}")
        print(response.text)

def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file="snapshot-data.json"):
    #create the snapshot url
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    while True:
        response = requests.get(snapshot_url, headers=headers)
        
        if response.status_code == 200:
            print("Snapshot is ready. Downloading...")
            snapshot_data = response.json()
            #write the snapshot to a new json file
            with open(output_file, "w", encoding="utf-8") as file:
                json.dump(snapshot_data, file, indent=4)
            print(f"Snapshot saved to {output_file}")
            break
        elif response.status_code == 202:
            print("Snapshot is not ready yet. Retrying in 10 seconds...")
        else:
            print(f"Error: {response.status_code}")
            print(response.text)
            break
        
        time.sleep(10)


if __name__ == "__main__":
    
    API_KEY = "your-bright-data-api-key"
    LOCATION = "Miami"
    CHECK_IN = "2025-02-01T00:00:00.000Z"
    CHECK_OUT = "2025-02-02T00:00:00.000Z"
    DATES = {
        "check_in": CHECK_IN,
        "check_out": CHECK_OUT
    }
    snapshot_id = get_bookings(API_KEY, LOCATION, DATES)
    poll_and_retrieve_snapshot(API_KEY, snapshot_id)
```

- `get_bookings()` は `API_KEY`、`LOCATION`、`DATES` を受け取ります。その後、データ要求のリクエストを行い、`snapshot_id` を返します。
- スナップショットを取得するには `snapshot_id` が必要です。
- `snapshot_id` が生成された後、`poll_and_retrieve_snapshot()` は 10 秒ごとにデータ準備状況を確認します。
- データの準備ができたら、`json.dump()` を使って JSON ファイルに保存します。

コードを実行すると、ターミナルには次のような表示が出るはずです。

```
Request successful. Response:
{
    "snapshot_id": "s_m5moyblm1wikx4ntot"
}
Polling snapshot for ID: s_m5moyblm1wikx4ntot...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is ready. Downloading...
Snapshot saved to snapshot-data.json
```

その後、次のようなオブジェクトが多数入った JSON ファイルが生成されます。

```json
{
        "input": {
            "url": "https://www.booking.com",
            "location": "Miami",
            "check_in": "2025-02-01T00:00:00.000Z",
            "check_out": "2025-02-02T00:00:00.000Z",
            "adults": 2,
            "rooms": 1
        },
        "url": "https://www.booking.com/hotel/us/ramada-plaze-by-wyndham-marco-polo-beach-resort.html?checkin=2025-02-01&checkout=2025-02-02&group_adults=2&no_rooms=1&group_children=",
        "location": "Miami",
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z",
        "adults": 2,
        "children": null,
        "rooms": 1,
        "id": "55989",
        "title": "Ramada Plaza by Wyndham Marco Polo Beach Resort",
        "address": "19201 Collins Avenue",
        "city": "Sunny Isles Beach (Florida)",
        "review_score": 6.2,
        "review_count": "1788",
        "image": "https://cf.bstatic.com/xdata/images/hotel/square600/414501733.webp?k=4c14cb1ec5373f40ee83d901f2dc9611bb0df76490f3673f94dfaae8a39988d8&o=",
        "final_price": 217,
        "original_price": 217,
        "currency": "USD",
        "tax_description": null,
        "nb_livingrooms": 0,
        "nb_kitchens": 0,
        "nb_bedrooms": 0,
        "nb_all_beds": 2,
        "full_location": {
            "description": "This is the straight-line distance on the map. Actual travel distance may vary.",
            "main_distance": "11.4 miles from downtown",
            "display_location": "Miami Beach",
            "beach_distance": "Beachfront",
            "nearby_beach_names": []
        },
        "no_prepayment": false,
        "free_cancellation": true,
        "property_sustainability": {
            "is_sustainable": false,
            "level_id": "L0",
            "facilities": [
                "436",
                "490",
                "492",
                "496",
                "506"
            ]
        },
        "timestamp": "2025-01-07T16:43:24.954Z"
    },
```

### AIOHTTP

[AIOHTTP](https://brightdata.jp/blog/web-data/speed-up-web-scraping) ライブラリを使用すると、複数のデータセットを同時にトリガー、ポーリング、ダウンロードできるため、このプロセスをより高速化できます。以下のコードは上記 Requests の例の概念を踏襲しつつ、`aiohttp.ClientSession()` を使って複数リクエストを非同期に実行します。

```python
import aiohttp
import asyncio
import json


async def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"
    dataset_id = "gd_m4bf7a917zfezv9d5"
    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    async with aiohttp.ClientSession(headers=headers) as session:
        async with session.post(endpoint, json=payload) as response:
            if response.status == 200:
                response_data = await response.json()
                print(f"Request successful for location: {location}. Response:")
                print(json.dumps(response_data, indent=4))
                return response_data["snapshot_id"]
            else:
                print(f"Error for location: {location}. Status: {response.status}")
                print(await response.text())
                return None


async def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file):
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    async with aiohttp.ClientSession(headers=headers) as session:
        while True:
            async with session.get(snapshot_url) as response:
                if response.status == 200:
                    print(f"Snapshot for {output_file} is ready. Downloading...")
                    snapshot_data = await response.json()
                    # Save snapshot data to a file
                    with open(output_file, "w", encoding="utf-8") as file:
                        json.dump(snapshot_data, file, indent=4)
                    print(f"Snapshot saved to {output_file}")
                    break
                elif response.status == 202:
                    print(f"Snapshot for {output_file} is not ready yet. Retrying in 10 seconds...")
                else:
                    print(f"Error polling snapshot for {output_file}. Status: {response.status}")
                    print(await response.text())
                    break

            await asyncio.sleep(10)


async def process_location(api_key, location, dates):
    snapshot_id = await get_bookings(api_key, location, dates)
    if snapshot_id:
        output_file = f"snapshot-{location.replace(' ', '_').lower()}.json"
        await poll_and_retrieve_snapshot(api_key, snapshot_id, output_file)


async def main():
    api_key = "your-bright-data-api-key"
    locations = ["Miami", "Key West"]
    dates = {
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z"
    }

    # Process all locations in parallel
    tasks = [process_location(api_key, location, dates) for location in locations]
    await asyncio.gather(*tasks)


if __name__ == "__main__":
    asyncio.run(main())
```

- `get_bookings()` と `poll_and_retrieve_snapshot()` は、いずれも `aiohttp.ClientSession` オブジェクトを使ってサーバーへ非同期リクエストを作成するようになりました。
- `process_location()` は、ある location に対する全データ処理に使用します。
- `main()` により、すべての location に対して同時に `process_location()` を呼び出せます。

出力例は以下です:

```
Request successful for location: Miami. Response:
{
    "snapshot_id": "s_m5mtmtv62hwhlpyazw"
}
Request successful for location: Key West. Response:
{
    "snapshot_id": "s_m5mtmtv72gkkgxvdid"
}
Polling snapshot for ID: s_m5mtmtv62hwhlpyazw...
Polling snapshot for ID: s_m5mtmtv72gkkgxvdid...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is ready. Downloading...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot saved to snapshot-miami.json
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is ready. Downloading...
Snapshot saved to snapshot-key_west.json
```

## Bright Data’s Alternative Solutions

[Web Scraper APIs](https://brightdata.jp/products/web-scraper) に加えて、Bright Data は多様なニーズに対応する、すぐに使えるデータセットも提供しています。特に需要の高い旅行系データセットには次のものがあります:

- [Hotel Datasets](https://brightdata.jp/products/datasets/travel/hotels)
- [Expedia Datasets](https://brightdata.jp/products/datasets/travel/expedia)
- [Tourism Datasets](https://brightdata.jp/products/datasets/tourism)
- [Booking.com Datasets](https://brightdata.jp/products/datasets/booking)
- [TripAdvisor Datasets](https://brightdata.jp/products/datasets/tripadvisor)

フルマネージドまたはセルフマネージドのカスタムデータセットを選択でき、あらゆる公開 Web サイトからデータを抽出して、要件どおりにカスタマイズできます。