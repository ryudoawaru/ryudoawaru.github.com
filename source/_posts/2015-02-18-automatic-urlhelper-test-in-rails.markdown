---
layout: post
title: "自動化測試 Rails 中的 URL HELPER"
date: 2015-02-18 12:51
comments: true
categories: [rails,rspec,route]
---

### 前言：

最近在某個和客戶共同開發的專案中，遇到了以下的情形：

1.	專案的 layout / view 經過大幅的改版
1.	改版是把舊 layout（以下簡稱 ver1）的大部份功能改出一份新版 layout / view / controller（以下稱 ver2）
1.	改版後幾乎沒有移除 ver1 重複的部份，因此系統中存著很多其實用不到的 view / controller / route
1.	又因為改版後不想改變在 production env 的 url，所以在路由設定中用了大概是類似的架構：
	-	Development 中有 /ver1 和 /ver2 兩個 route namespace，而且有 base uri，意即可能會有 /ver1/products 和 /ver2/products
	-	Production 中，這兩個 namespace 的 base uri 會被消除，舉例來說，如果遇到 /products 的 request，系統會先找 /ver2 有沒有 /products，有的話會先使用，沒有再去找 /ver1

所以系統最後就存在一堆功能重疊的 controller / view / layout 了，現在的工作是要移除 /ver1 和 /ver2 中重複的部份。

### 執行：

基本上就是高科技手工業，程序如下：

1.	找 /ver2 的 route，假設現在的目標是 resources :users
1.	在 /ver1 上找尋類似的 resource
1.	如果有，就把 /ver1 的刪掉
1.	如果不是單純的 Restful CRUD（意指有其它非 index. show, create 等標準 action 的 action）就得要一一檢查在 /ver2 上是否都有相對應的 action
1.	找到所有的 view / helper / asset / controller 中是否有用到 /ver1 上的 url helper method，然後統一取代

以上程序有效的前提是，所有的 URL 都用 helper method 而不用純字串或 url_for 等。

事情聽起來好像不是很難，不過這個專案可怕的地方就在於數百行的 route 中幾乎沒有單純 CRUD 的 resource，所以和同事花上了很多時間在手工上，完成後的問題就是，要怎樣確保沒有遺漏的地方。

於是想到，是否可以將程式中所有用到的 url helper method 集合起來一一做檢查呢？最後想到的解法大意如下：

1.	用文字搜尋的程式，以正規表示式去找出程式碼中使用的 url helper method
1.	將找到的 method 集合產生列表
1.	用列表產生 it 區塊
1.	it 區塊中使用 [helper spec](https://www.relishapp.com/rspec/rspec-rails/v/2-4/docs/helper-specs/helper-spec) 去測試 view context 中是否有這些 method

文字搜尋採用目前比較流行的 [ack](http://beyondgrep.com/)，好處是除了速度快之外又是 Perl 版的 Regexp，比 egrep 內建的更貼近 Ruby 使用的版本

```bash
ack --noheading -h '[^A-Za-z0-9_.].((ver1|ver2)[\\w_]+?\_(url|path))' app/ --output '$1' | sort | uniq
```

使用的 ack 參數如下：

*	正規表示式為：```[^A-Za-z0-9_.].((ver1|ver2)[\\w_]+?\_(url|path))```
*	--noheading 不顯示檔案名在頭
*	-h 不顯示檔名在左邊
*	--output '$1' 只顯示批配的第 1 項

測試的程式大概長的像這樣：

```ruby
RSpec.describe ApplicationHelper, :type => :helper do
  fn = "tmp/url_helpers.txt"
  cmd = "ack --noheading -h '[^A-Za-z0-9_.].((ver1|ver2)[\\w_]+?\_(url|path))' app/ --output '$1' | sort | uniq > #{fn}"
  system cmd
  File.read(fn).split.each do |url_helper_name|
    it "Url helper #{url_helper_name} should be available" do
      helper.respond_to?(url_helper_name.to_sym).should be(true)
    end
  end
end
```

這樣一跑完就可以放心了。