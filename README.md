# 系統設計

## Tinder System Design

this note is based on the coursing video
[Tinder Microservices Architecture | Online Dating App System Design](https://youtu.be/XFQIW2R_Klk)
5000萬user

160萬swipes per day

### function requirement

1. user 帳號 第三方登入
2. 每人五張照片
3. 看到附近用戶的搜尋標準
    1. 距離
    2. 性別
    3. 年齡範圍
    4. 種族
    5. 興趣
4. 左右滑來喜歡不喜歡
5. 三個月內不顯示重複看到的用戶
6. 配對→聊天 推播給雙方
7. 分析與監看
    1. 任何分散式系統(任何系統)都需要分析與監看
    2. 功能和產品deliver 都需要
8. 視訊聲音通話

### non functional requirement

1. 高可用性
    1. 若機器/節點 服務壞掉，只有非常少用戶受到影響，影響時間也要盡可能降低
2. 高擴展性
3. 低延遲
    1. 顯示用戶資訊
    2. 左右滑體驗

1. 個人資料持久化
2. 一致性，強一致或最終一致？

### API Spec

1. 登入
    1. 第三方登入
    2. 送出SMS 手機驗證碼以做驗證
2. 上傳 以及 更新用戶資料（圖片, 介紹等）
    1. user token
    2. 資料
3. 搜尋喜好API 
    1. user token 
    2. 搜尋喜好
4. 取得附近用戶API （getProfile）
    1. user token 
    2. counts, 附近用戶數量
5. rateProfiles (user token, profiles rating)
    1. 不必每次滑都call
    2. 可儲存10~20次滑動喜好後再call api
    3. 是否需要儲存 dislike?

### 架構

![截圖 2022-12-25 上午3.34.14.png](%E7%B3%BB%E7%B5%B1%E8%A8%AD%E8%A8%88%205092427a77d04d7d81a4869d395bd25d/%25E6%2588%25AA%25E5%259C%2596_2022-12-25_%25E4%25B8%258A%25E5%258D%25883.34.14.png)

1. Gateway/ routing service 
2. Analytics
3. Chat service → Push notification
4. Swipe service → Push notifation
5. Recommendation service
6. Profile service 
7. User Registration → SMS gateway

用戶註冊服務

個人資料由個人資料服務創建儲存

推薦系統，基於搜尋喜好顯示附近用戶

滑動的資訊進入swipe service

matched 觸發 聊天服務

<aside>
💡 客戶端會和gateway 使用websocket 或 http 長輪詢

Long polling 長輪詢：發送req 收到req之後，如果有資料就斷開，斷開後client 再發一次req

</aside>

假設使用websocket 和 Gateway 溝通

- 聊天服務
- 滑動服務
- 推薦服務等都會用到

**個人資料服務**

1. 儲存個人資料
    1. bio
    2. 年齡 性別 個人圖片
    3. 用戶位置
    4. 用戶喜好
2. user table, sharded by user id

| PK | User ID string(16) |
| --- | --- |
|  | Age int |
|  | gender varchar |
|  | email varchar |
|  | phone string |
|  | interest string(4096) |
|  |  |

如何儲存用戶圖片

三種方式：

1. **儲存在user table** 
    1. 新增欄位放blob
    2. 新增image table, userID 當key, image ID 以及儲存 blob欄位
2. **儲存在 file system** 
    1. 儲存 檔案位置在user table 
3. **儲存在物件儲存服務 (Object storage service)**

第一種方式：

- 優點
    1. 方便好實作，彈性好
    2. 次要複本由資料庫管理，管理方便
    3. 不會有孤兒檔案
- 缺點
    1. 擴展性低
    2. 每次複製需要花費大量時間，製作次要複本要很久
    3. 圖片blob 非常大
    4. PB級 百萬張圖片 的複製非常昂貴
    5. 不論sql nosql都將資料儲存在區塊，如果圖片大於區塊就會分到不同區塊中，取圖片將耗費更多時間，不論讀寫都會非常昂貴
    6. 存取大型blob時，也會影響其他IO, 資料庫操作的效能

第二種方式：

- 優點
    1. 只需要儲存檔案位置在資料庫
    2. commit log 會很小
    3. 檔案傳輸更快
- 缺點
    1. 需要負責檔案備份
    2. 上傳或刪除檔案，如果出錯，會產生懸垂檔案（永遠不會被發現）
    3. 需要自己shard 
    4. 取得大量檔案速度會很慢，多次disk IO
    5. 多數作業系統會限制檔案儲存上限，數量一多會產生很多folder 和子folder 緩慢！

第三種方法：

- 優點
    1. 儲存在物件儲存服務，如：S3
    2. 只需要存URI 在 user table 
    3. 檔案、圖片、影片以物件形式儲存
    4. 可以快取系統整合，如：CDN
- 缺點

儲存圖片總結來說需要考慮：

1. Mutability
2. Transaction guarantee (ACID)
3. Indexes (search)
4. Access control 

User Profile 服務

![截圖 2022-12-25 上午4.39.14.png](%E7%B3%BB%E7%B5%B1%E8%A8%AD%E8%A8%88%205092427a77d04d7d81a4869d395bd25d/%25E6%2588%25AA%25E5%259C%2596_2022-12-25_%25E4%25B8%258A%25E5%258D%25884.39.14.png)

至少需要三個server，如果有任何一個掛了，還有一個運作以及一個備援，由負載平衡器分配工作

**Swipe service** 

將十幾次的滑動儲存在快取，並在一段時間後將結果傳給swipe service

滑動結果table 

| PK | UserID1, UserID2 string(16) |
| --- | --- |
|  | created time datetime |
|  | user1 liked boolean |
|  | user2 liked boolean |

滑動結果儲存時常需多久？三個月一年？

### 推薦系統

- hard preferences (必符合user 要求)
    - 性別
    - 距離
    - 年齡區間
- soft preference
    - 興趣(看書 電影)
    - 喜好 (運動 健身)

使用者活躍度越高，個人排名和出現機率越高

推薦系統資料儲存方式:Geosharded

- 使用Geosharded 儲存，由使用者地理位置分片，uber 的使用者地理位置會持續變動，但是 Tinder 使用者地理位置並不會持續變動，地理位置max size 可根據使用者設定的範圍喜好設定
- 在美國的伺服器高峰時期，在印度的伺服器會在低谷，因此為提高機器使用率以降低成本，可以map 不同國家的地理分片到同一台實體機器上，讓機器不會有閒置低峰期。（但這樣實體機器位置要在…?）
- 有些地方會要求使用者的資料需存放在相同區域，EU 規定使用者資料需儲存在歐洲區域

**user profile table**

|  | User Profile |  |
| --- | --- | --- |
| PK | UserID string(16) |  |
| SK(二鍵) | CurrentGrid string(64) | sharded key |
|  | PreviousGrid string(64) |  |
| SK | LastLoginTime datetime |  |
|  | Age int |  |
|  | Gender string(6) |  |
|  | Interest string(4096) | json string |

**user search result** 

| PK | UserID string(16) |
| --- | --- |
|  | LastoginTime datetime |
|  | SearchResult string(4096) |
|  | xxx:xxx |

為加快使用者滑動時，取得資料，可以將多個使用者的serach query 做成快取，這樣同個地理分片內的使用者都能使用**快取**來取得推薦

推薦系統架構：

![截圖 2022-12-25 下午5.38.41.png](%E7%B3%BB%E7%B5%B1%E8%A8%AD%E8%A8%88%205092427a77d04d7d81a4869d395bd25d/%25E6%2588%25AA%25E5%259C%2596_2022-12-25_%25E4%25B8%258B%25E5%258D%25885.38.41.png)

Search Engines 可以為不同使用者 rank 推薦，可以使用機器學習的算法建構推薦系統

1. Content based 內容為基礎
2. Collaborative Filtering 協同過濾

Contant based 將相似的屬性歸類，找出相似的使用者；而協同過濾的方式則是透過眾人的意見進行分析，例如屬性和A user 相同的user 都滑user B like，那user A很可能就會喜歡 userB，以推薦更個性化的結果