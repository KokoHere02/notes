# stream-lite 项目

## 技术栈 (后端)
- jdk17
- spring boot3
- mybatis-flex
- PostgreSQL

## 数据库表
`users`（用户表）
`rooms`（直播间表）
`streams`（推流记录）
`messages`（直播间聊天消息）
`follows`（关注关系）
`room_records`（直播历史记录，开播/下播统计）

```sql 
  -- ========== users ==========
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,            -- bcrypt 或其他算法加密后的密码
    password_salt VARCHAR(32),                -- 可选，记录盐值
    email VARCHAR(100) UNIQUE,                -- 可选邮箱登录
    phone VARCHAR(20) UNIQUE,                 -- 可选手机号登录
    avatar_url VARCHAR(255),                   -- 存储头像URL或ID
    bio TEXT,
    status SMALLINT NOT NULL DEFAULT 1,       -- 1=正常, 0=禁用, 2=封禁
    last_login TIMESTAMP,                     -- 最近登录时间
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 自动更新时间 trigger（PostgreSQL）
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
   NEW.updated_at = NOW();
   RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_updated_at();

-- ========== live_rooms ==========
CREATE TABLE live_rooms (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(100) NOT NULL,
    cover_url TEXT,
    status SMALLINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_live_room_user FOREIGN KEY (user_id) REFERENCES users(id)
);

COMMENT ON TABLE live_rooms IS '直播间';
COMMENT ON COLUMN live_rooms.user_id IS '主播用户ID';
COMMENT ON COLUMN live_rooms.title IS '直播间标题';
COMMENT ON COLUMN live_rooms.cover_url IS '直播间封面图';
COMMENT ON COLUMN live_rooms.status IS '直播状态：0 未开播，1 直播中，2 已结束';
COMMENT ON COLUMN live_rooms.created_at IS '创建时间';
COMMENT ON COLUMN live_rooms.updated_at IS '更新时间';


-- ========== streams ==========
CREATE TABLE streams (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT NOT NULL,
    stream_key VARCHAR(100) NOT NULL,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    bitrate INT,
    fps INT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_stream_room FOREIGN KEY (room_id) REFERENCES live_rooms(id)
);

COMMENT ON TABLE streams IS '直播流记录';
COMMENT ON COLUMN streams.room_id IS '直播间ID';
COMMENT ON COLUMN streams.stream_key IS '推流密钥';
COMMENT ON COLUMN streams.start_time IS '开播时间';
COMMENT ON COLUMN streams.end_time IS '下播时间';
COMMENT ON COLUMN streams.bitrate IS '码率（kbps）';
COMMENT ON COLUMN streams.fps IS '帧率';
COMMENT ON COLUMN streams.created_at IS '记录创建时间';


-- ========== messages ==========
CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_message_room FOREIGN KEY (room_id) REFERENCES live_rooms(id),
    CONSTRAINT fk_message_user FOREIGN KEY (user_id) REFERENCES users(id)
);

COMMENT ON TABLE messages IS '直播间消息';
COMMENT ON COLUMN messages.room_id IS '直播间ID';
COMMENT ON COLUMN messages.user_id IS '发送消息的用户ID';
COMMENT ON COLUMN messages.content IS '消息内容';
COMMENT ON COLUMN messages.created_at IS '发送时间';


-- ========== follows ==========
CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    target_user_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_follow_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_follow_target FOREIGN KEY (target_user_id) REFERENCES users(id),
    CONSTRAINT uq_follow UNIQUE (user_id, target_user_id)
);

COMMENT ON TABLE follows IS '关注关系';
COMMENT ON COLUMN follows.user_id IS '粉丝用户ID';
COMMENT ON COLUMN follows.target_user_id IS '被关注的主播ID';
COMMENT ON COLUMN follows.created_at IS '关注时间';


-- ========== room_records ==========
CREATE TABLE room_records (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT NOT NULL,
    stream_id BIGINT NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    total_viewers INT DEFAULT 0,
    max_online INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_record_room FOREIGN KEY (room_id) REFERENCES live_rooms(id),
    CONSTRAINT fk_record_stream FOREIGN KEY (stream_id) REFERENCES streams(id)
);

COMMENT ON TABLE room_records IS '直播间历史记录';
COMMENT ON COLUMN room_records.room_id IS '直播间ID';
COMMENT ON COLUMN room_records.stream_id IS '关联的直播记录ID';
COMMENT ON COLUMN room_records.start_time IS '开播时间';
COMMENT ON COLUMN room_records.end_time IS '下播时间';
COMMENT ON COLUMN room_records.total_viewers IS '累计观看人数';
COMMENT ON COLUMN room_records.max_online IS '最高同时在线人数';
COMMENT ON COLUMN room_records.created_at IS '创建时间';


```

## 引入JWT
```xml 
  <!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-api -->
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>${jjwt.version}</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/io.jsonwebtoken/jjwt-impl -->
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>${jjwt.version}</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>${jjwt.version}</version>
  </dependency>
```
### JWTUtil
```java 

public class JwtUtil {
  private JwtUtil() {
  }

  private static final String JWT_KEY = SpringUtil.getProperty("crypto.jwt-key","");
  private static final long EXPIRATION = 3600_000; // 1小时

  public static String generateToken(@NonNull String id) {
    return Jwts.builder()
      .subject(id)
      .issuedAt(new Date())
      .expiration(new Date(System.currentTimeMillis() + EXPIRATION))
      .signWith(getSigningKey())
      .compact();
  }


  private static SecretKey getSigningKey() {
    byte[] keyBytes = Decoders.BASE64.decode(JWT_KEY);
    return Keys.hmacShaKeyFor(keyBytes);
  }

}
```
## Login 
```java 

  @Override
  public String login(@NonNull UserDto user) {
    String username = user.getUsername();
    String password = user.getPassword();
    if (!validUsername(username) || !validPassword(password)) {
      // todo 抛出错误 非法参数
      return null;
    }
    String decryptPassword = CryptoUtil.rsaDecrypt(password);
    Users dbUser = this.queryChain()
      .eq(Users::getUsername, username)
      .one();
    if  (dbUser == null) {
      // todo 抛出错误 用户不存在
      return null;
    }
    String decrypt = CryptoUtil.aesDecrypt(dbUser.getPassword());
    if (decrypt.isBlank() || !decrypt.equals(decryptPassword)) {
      // todo 抛出错误 密码错误
      return null;
    }

    return JwtUtil.generateToken(dbUser.getId().toString());
  }

```

# stream-lite-web

## 技术栈（前端）
- vue3
- vite
- route
- pinia
- tailwin3
- DaisyUI + Headless UI


## 头部组件（暂定）

```html
<div class="navbar bg-base-100 shadow-sm px-4">
  <!-- 左侧 -->
  <div class="navbar-start">
    <a
      class="btn btn-ghost text-xl
             text-base-content
             hover:bg-primary/15
             hover:text-base-content"
    >
      daisyUI
    </a>
  </div>

  <!-- 中间搜索 -->
  <div class="navbar-center">
    <label class="input input-bordered flex items-center gap-2 w-80">
      <svg
        class="h-[1em] opacity-50"
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 24 24"
      >
        <g
          stroke-linejoin="round"
          stroke-linecap="round"
          stroke-width="2.5"
          fill="none"
          stroke="currentColor"
        >
          <circle cx="11" cy="11" r="8" />
          <path d="m21 21-4.3-4.3" />
        </g>
      </svg>

      <input
        v-model="query"
        @keydown.enter="submitSearch"
        type="search"
        class="grow"
        placeholder="Search"
      />

      <kbd class="kbd kbd-sm">⌘</kbd>
      <kbd class="kbd kbd-sm">K</kbd>
    </label>
  </div>
```
## 卡片组件
```html
  <!-- 右侧操作区 -->
  <div class="main">
  <div class="card bg-base-100 w-96 shadow-sm">
    <figure>
      <img
        src="https://img.daisyui.com/images/stock/photo-1606107557195-0e29a4b5b4aa.webp"
        alt="Shoes"
      />
    </figure>

    <div class="card-body gap-2 px-3 py-3">
      <p>4点世纪大战 HLE vs T1</p>

      <div class="flex items-center justify-between">
        <div class="flex items-center">
          <img
            class="w-8 h-8 rounded-full"
            src="https://img.daisyui.com/images/profile/demo/yellingcat@192.webp"
          />
          <span class="ml-2 text-sm font-medium">Card Title</span>
        </div>

        <span class="text-xs text-gray-500">200 人</span>
      </div>
    </div>
  </div>
</div>

```