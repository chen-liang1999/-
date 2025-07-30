import streamlit as st
import pandas as pd
import re
from playwright.sync_api import sync_playwright
import requests

st.set_page_config(page_title="小程序数据抓取工具", layout="wide")
st.title("📦 微信小程序商品与销售数据抓取工具")

h5_url = st.text_input("请输入小程序 H5 页面链接（来自公众号或分享）:")

if st.button("开始抓取"):
    if not h5_url:
        st.warning("⚠️ 请输入有效的小程序 H5 链接")
    else:
        with sync_playwright() as p:
            browser = p.chromium.launch(headless=False)
            context = browser.new_context()
            page = context.new_page()

            api_urls = []

            def handle_response(response):
                url = response.url
                if any(k in url for k in ["goods", "product", "order"]):
                    st.write(f"🔎 捕获 API: {url}")
                    api_urls.append(url)

            page.on("response", handle_response)
            page.goto(h5_url)

            st.info("👉 请在新打开的浏览器窗口扫码登录并进入商品页，然后回到这里点击“继续抓取”")
            if st.button("继续抓取"):
                cookies = context.cookies()
                cookie_str = "; ".join([f"{c['name']}={c['value']}" for c in cookies])
                st.success("✅ 登录成功，Cookie 已获取")

                if not api_urls:
                    st.error("⚠️ 没有捕获到相关 API，请确认已进入商品页")
                else:
                    goods_api = [url for url in api_urls if "list" in url]
                    if not goods_api:
                        st.error("⚠️ 没有找到商品列表 API")
                    else:
                        goods_api = goods_api[0]
                        st.write(f"📌 选用商品列表 API: {goods_api}")

                        headers = {"Cookie": cookie_str, "User-Agent": "MicroMessenger"}
                        resp = requests.get(goods_api, headers=headers)
                        goods_list = resp.json().get("data", {}).get("goods_list", [])

                        enriched_goods = []
                        for g in goods_list:
                            detail_api = re.sub(r"list.*", f"detail?id={g['id']}", goods_api)
                            detail_resp = requests.get(detail_api, headers=headers).json()
                            detail = detail_resp.get("data", {})
                            enriched_goods.append({
                                "商品ID": g["id"],
                                "商品名称": g["name"],
                                "价格": g["price"],
                                "库存": detail.get("stock", "未知"),
                                "销量": detail.get("sales", "未知")
                            })

                        df = pd.DataFrame(enriched_goods)
                        st.dataframe(df)
                        csv = df.to_csv(index=False).encode("utf-8-sig")
                        st.download_button("⬇️ 下载商品数据 CSV", csv, "goods_data.csv", "text/csv")
