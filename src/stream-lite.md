# stream-lite 项目

## 技术栈
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
    password VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    bio TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE users IS '用户表';
COMMENT ON COLUMN users.username IS '用户名';
COMMENT ON COLUMN users.password IS '密码（加密后）';
COMMENT ON COLUMN users.avatar_url IS '头像 URL';
COMMENT ON COLUMN users.bio IS '个人简介';
COMMENT ON COLUMN users.created_at IS '创建时间';
COMMENT ON COLUMN users.updated_at IS '更新时间';


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