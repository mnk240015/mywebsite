import os
import re
import urllib.parse
from bs4 import BeautifulSoup
import pandas as pd
import requests


def fetch_yahoo_realtime_account(username):
    """Yahoo!リアルタイム検索から特定アカウントの最新ツイートを取得する"""
    # 「from:アカウント名」で検索することで、その人のツイートのみに絞り込む
    search_query = f"from:{username}"
    encoded_query = urllib.parse.quote(search_query)
    url = f"https://search.yahoo.co.jp/realtime/search?p={encoded_query}"

    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
    }

    try:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
    except Exception as e:
        print(f"アクセスに失敗しました: {e}")
        return []

    soup = BeautifulSoup(response.text, "html.parser")
    tweet_data = []

    # 各ツイートを囲む要素（articleタグ）を抽出
    articles = soup.find_all("article")
    if not articles:
        articles = soup.find_all(
            "div", class_=lambda c: c and "Tweet_tweet" in c
        )

    for article in articles:
        try:
            # 1. ツイートの個別URLを取得
            link_elem = article.find(
                "a", href=re.compile(r"twitter\.com/.+/status/")
            ) or article.find("a", href=re.compile(r"x\.com/.+/status/"))
            if not link_elem:
                continue
            tweet_url = link_elem["href"]

            # 2. 本文の取得
            text_elem = article.find(
                "p"
            ) or article.find("div", class_=lambda c: c and "Tweet_body" in c)
            text = (
                text_elem.get_text(strip=True)
                if text_elem
                else "本文の取得失敗"
            )

            # 3. 投稿日時の取得（「〇分前」などの文字列）
            time_elem = article.find("time") or article.find(
                "span", class_=lambda c: c and "Tweet_time" in c
            )
            time_str = (
                time_elem.get_text(strip=True) if time_elem else "日時不明"
            )

            tweet_data.append(
                {"tweet_url": tweet_url, "created_at": time_str, "text": text}
            )
        except Exception:
            continue

    return tweet_data


def update_tweet_database(username, file_path="tweets_data.csv"):
    """既存のCSVを確認し、重複のない新しいデータのみを追記する"""
    # 1. 最新データのスクレイピング
    new_tweets = fetch_yahoo_realtime_account(username)
    if not new_tweets:
        print(
            f"@{username} の新規データが取得できなかったか、ツイートがありません。"
        )
        return

    df_new = pd.DataFrame(new_tweets)

    # 2. 既存のCSVファイルを読み込む（古いデータを削除しない）
    if os.path.exists(file_path):
        df_old = pd.read_csv(file_path)
    else:
        df_old = pd.DataFrame(columns=["tweet_url", "created_at", "text"])

    # 3. 重複チェック（tweet_urlをキーにする）
    if not df_old.empty:
        existing_urls = set(df_old["tweet_url"].astype(str))
        # 既存リストにないURL（＝新しいツイート）だけを抽出（重複は更新しない）
        df_to_add = df_new[~df_new["tweet_url"].isin(existing_urls)]
    else:
        df_to_add = df_new

    # 4. データの結合と保存
    if not df_to_add.empty:
        df_combined = pd.concat([df_old, df_to_add], ignore_index=True)
        df_combined.to_csv(file_path, index=False, encoding="utf-8-sig")
        print(f"新たに {len(df_to_add)} 件のツイートを追加しました。")
    else:
        print("新しいツイートはありませんでした（すべて重複）。")


if __name__ == "__main__":
    # 取得したいアカウントのID（@マークの後ろの英数字）を指定
    # 例: @GitHub のツイートを取りたい場合は "GitHub" と書きます
    USER_NAME = "PFPnews"

    update_tweet_database(USER_NAME)
