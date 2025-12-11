# stream-lite 项目

`users`（用户表）
`rooms`（直播间表）
`streams`（推流记录）
`messages`（直播间聊天消息）
`follows`（关注关系）
`room_records`（直播历史记录，开播/下播统计）

```sql 
  CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE COMMENT '用户名',
    password VARCHAR(255) NOT NULL COMMENT '密码（加密后）',
    avatar_url TEXT COMMENT '头像 URL',
    bio TEXT COMMENT '个人简介',
    created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '创建时间',
    updated_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '更新时间'
  );

    CREATE TABLE live_rooms (
      id BIGSERIAL PRIMARY KEY,
      user_id BIGINT NOT NULL COMMENT '主播用户ID',
      title VARCHAR(100) NOT NULL COMMENT '直播间标题',
      cover_url TEXT COMMENT '直播间封面图',
      status SMALLINT NOT NULL DEFAULT 0 COMMENT '直播状态：0 未开播，1 直播中，2 已结束',
      created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '创建时间',
      updated_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '更新时间',

      CONSTRAINT fk_live_room_user FOREIGN KEY (user_id) REFERENCES users(id)
  );

    CREATE TABLE streams (
      id BIGSERIAL PRIMARY KEY,
      room_id BIGINT NOT NULL COMMENT '直播间ID',
      stream_key VARCHAR(100) NOT NULL COMMENT '推流密钥',
      start_time TIMESTAMP COMMENT '开播时间',
      end_time TIMESTAMP COMMENT '下播时间',
      bitrate INT COMMENT '码率（kbps）',
      fps INT COMMENT '帧率',
      created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '记录创建时间',

      CONSTRAINT fk_stream_room FOREIGN KEY (room_id) REFERENCES live_rooms(id)
  );

  CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT NOT NULL COMMENT '直播间ID',
    user_id BIGINT NOT NULL COMMENT '发送消息的用户ID',
    content TEXT NOT NULL COMMENT '消息内容',
    created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '发送时间',

    CONSTRAINT fk_message_room FOREIGN KEY (room_id) REFERENCES live_rooms(id),
    CONSTRAINT fk_message_user FOREIGN KEY (user_id) REFERENCES users(id)
  );

  CREATE TABLE follows (
      id BIGSERIAL PRIMARY KEY,
      user_id BIGINT NOT NULL COMMENT '粉丝用户ID',
      target_user_id BIGINT NOT NULL COMMENT '被关注的主播ID',
      created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '关注时间',

      CONSTRAINT fk_follow_user FOREIGN KEY (user_id) REFERENCES users(id),
      CONSTRAINT fk_follow_target FOREIGN KEY (target_user_id) REFERENCES users(id),
      CONSTRAINT uq_follow UNIQUE (user_id, target_user_id)
  );

  CREATE TABLE room_records (
      id BIGSERIAL PRIMARY KEY,
      room_id BIGINT NOT NULL COMMENT '直播间ID',
      stream_id BIGINT NOT NULL COMMENT '关联的直播记录ID',
      start_time TIMESTAMP NOT NULL COMMENT '开播时间',
      end_time TIMESTAMP COMMENT '下播时间',
      total_viewers INT DEFAULT 0 COMMENT '累计观看人数',
      max_online INT DEFAULT 0 COMMENT '最高同时在线人数',
      created_at TIMESTAMP NOT NULL DEFAULT NOW() COMMENT '创建时间',

      CONSTRAINT fk_record_room FOREIGN KEY (room_id) REFERENCES live_rooms(id),
      CONSTRAINT fk_record_stream FOREIGN KEY (stream_id) REFERENCES streams(id)
  );



```