---
title: Next.js 9.3과 Prisma로 정적 웹사이트 만들기
date: 2020-12-03 21:12:38
category: Development
draft: false
---

> 이 글은 [Static Sites with Next.js 9.3 and Prisma - Lee Robinson](https://leerob.io/blog/next-prisma) 를 번역한 글 입니다.
> 잘못 번역한 부분이 있다면 댓글로 알려주세요!

정적 사이트는 현재 인기가 있고, 그럴만한 이유도 있다.

[JAMstack](https://jamstack.org/)은 기존 웹 앱의 매력적인 대안으로 등장했다. 정적 웹 앱은 다양한 장점이 있다.

- 큰 확장성
- 파일 호스팅 비용 절감
- CDN에서 캐싱된 에셋을 제공하여 더 빠른 성능
- 서버나 DB의 취약점에 대해 걱정 불필요
- 보다 향상된 개발자 경험 - 복잡하지 않은 배포 프로세스

만약 동적인 부분이 필요하다면 어떻게 해야할까? Next.js 9.3으로 해결할 수 있다.

# Next.js 9.3

9.3 이전에는 [Next.js](https://nextjs.org/)는 주로 서버 사이드 렌더링 방식을 사용하는 리액트 앱에서 사용되었다. 이는 검색 엔진 최적화(SEO), 라우팅, 코드 스플리팅을 포함한 대부분의 클라이언트 사이드 렌더링을 사용하는 리액트 앱에서 발생하는 몇몇 이슈들을 해결했다. 하지만 한가지 문제가 있다. 서버가 필요하다는 것이다. 이는 당신의 배포 옵션에 제한이 생기게 하고, 비용을 증가시킨다. Next.js 9.3 버전이 나오게 된 이유이다.

9.3 릴리즈 버전에서 Next.js는 완벽한 정적 웹사이트 생성기가 되었다. 새로 나온 getStaticProps 와 getStaticPaths 2개의 함수를 사용하면, 우리는 동적 데이터를 가져와서 정적 사이트를 생성할 수 있다.

특별한 점이 있다면 getStaticProps는 빌드할 때만 실행된다는 점이다. 심지어 이는 브라우저 번들에도 포함되어 있지 않다. 이는 당신이 직접 DB 쿼리를 포함한 서버측 코드를 작성할 수 있다는 것이다. 이제 Prisma를 시작해보자.

# Prisma

[Prisma](https://www.prisma.io/)는 데이터베이스에 접속하기 쉽게 해준다. 데이터베이스 스키마를 기반으로 자동 생성되고 형식이 안전한 쿼리를 통해 이전보다 쉽게 데이터를 관리 할 수 있다. 현재 MySQL, SQLite와 Postgres를 지원한다. 이번 튜토리얼에선 SQLite를 사용한다.

Prisma와 Next.js를 결합하면 Create/Update/Delete API 코드를 작성하지 않고도 데이터베이스와 직접 통신할 수 있다.

# Getting Started

특징은 충분히 전달했으니 이제 한번 시작해보자. [결과](https://prisma-next.now.sh)

- 모든 사이트는 정적으로 빌드되었으며, 어디서든 호스팅될 수 있다.
- 데이터는 빌드 시에 읽어진다.
- 페이지는 동적으로 생성되어진다. (예를 들어 `/songs/[id]`)
  ![](https://images.velog.io/images/sirl/post/3c59f912-04da-4e8e-8d93-97dc1c4e1213/prisma.gif)
  진행하기 전에, Node 10 이상 버전을 설치해야 한다!

# Download Starter

시간을 절약하기 위해, 나는 이미 우리가 튜토리얼을 진행하는데 필요한 스키마와 DB 설정을 마쳐두었다. 설명해줄테니 걱정하지 마라. 우선 레포지토리를 클론받고, 의존성 설치를 진행해라.

```
$ git clone https://github.com/leerob/next-prisma-starter.git
$ cd next-prisma-starter
$ yarn
```

몇몇 중요한 파일이 포함되어 있다.

- prisma/schema.prisma : DB 모델을 정의하는 Prisma 파일.
- prisma/dev.db : 노래와 가수 정보가 저장된 Sqlite 파일.

# Prisma Schema & Database

우리의 DB 스키마를 한번 확인해보자.

우리는 Song 과 Artist 라는 두 개의 모델이 있다. Song은 name이나 Artist와의 관계 같은 필요한 정보를 지니고 있다. 이 튜토리얼에선 Song은 단 하나의 Artist를 갖는다.

prisma/schema.prisma

```
datasource db {
  provider = "sqlite"
  url      = "file:dev.db"
}

generator client {
  provider = "prisma-client-js"
}

model Song {
  id            Int     @default(autoincrement()) @id
  name          String
  artist        Artist? @relation(fields: [artistId], references: [id])
  artistId      Int?
}

model Artist {
  id    Int     @default(autoincrement()) @id
  name  String  @unique
  songs Song[]
}
```

나는 dev.db에 song과 artist 데이터를 넣어두었다.

![](https://images.velog.io/images/sirl/post/932af7db-48a4-4b5e-a5da-ac7b2f0a174a/prisma1.PNG)

# Querying Data

우리는 데이터베이스에 샘플 데이터들을 넣어두었다. 이제 모든 노래를 보여주는 랜딩페이지를 만들어보자. `pages/index.js`로 이동해보자. 이 파일이 애플리케이션의 Entry point 이다.

이 파일은 `getStaticProps`를 통해 데이터베이스에서 노래를 쿼리 할 수 있도록 해준다. 그런 다음 결과를 반복해서 리스트로 표시해준다.

pages/index.js

```
export async function getStaticProps() {
  return {
    props: {
      songs: [
        {
          id: 1,
          name: 'Test Song'
        }
      ]
    }
  };
}

export default ({ songs }) => (
  <ul>
    {songs.map((song) => (
      <li key={song.id}>{song.name}</li>
    ))}
  </ul>
);
```

이제 실제 데이터를 표시해보도록 하자. Prisma Client를 추가한 뒤,`findMany` 함수를 사용하여 모든 song 데이터를 가져오도록 한다. Prisma의 가장 좋은 기능 중 하나는 관계와 관련된 작업을 쉽게 해준다는 것이다. `include` 를 통해 Song에 알맞은 Artist 정보를 불러올 수 있다.

pages.index/js

```
import { PrismaClient } from '@prisma/client';

export async function getStaticProps() {
  const prisma = new PrismaClient();
  const songs = await prisma.song.findMany({
    include: { artist: true }
  });

  return {
    props: {
      songs
    }
  };
}

export default ({ songs }) => (
  <ul>
    {songs.map((song) => (
      <li key={song.id}>{song.name}</li>
    ))}
  </ul>
);
```

Next.js가 이 페이지를 컴파일할 때, 데이터베이스에 쿼리를 날려 모든 song을 리스트로 보여줄 것 이다.

# Database Migrations & Editing Data

노래 목록도 멋지지만, 더 많은 것을 원하지 않는가? 우리가 각각의 노래에 유튜브에서 가져온 뮤직비디오를 추가하고 싶다고 해보자. 이는 데이터베이스 수정이 필요할 것이다. 걱정하지마라 - Prisma가 도와줄 것이다.

[Prisma Migrate](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-migrate)는 데이터베이스 스키마를 변경할 수 있도록 돕는 도구이다. 예를 들어, 테이블을 추가하거나 기존 테이블에 칼럼을 추가할 수 있다. 이러한 변경들은 _스키마 마이그레이션_ 이라고 한다.

Prisma Migrate는 아직 실험적이지만, 곧 배포 준비가 될 예정이다. 그동안은 일반 SQL이나 다른 마이그레이션 툴을 사용해도 좋다.

우리의 스키마에 새로운 필드를 추가해보자.

prisma/schema.prisma

```
datasource db {
  provider = "sqlite"
  url      = "file:dev.db"
}

generator client {
  provider = "prisma-client-js"
}

model Song {
  id            Int     @default(autoincrement()) @id
  name          String
  youtubeId     String?
  albumCoverUrl String?
  artist        Artist? @relation(fields: [artistId], references: [id])
  artistId      Int?
}

model Artist {
  id    Int     @default(autoincrement()) @id
  name  String  @unique
  songs Song[]
}
```

마이그레이션을 생성하기 위해선 하단의 코드를 실행해야 한다.

```
npx prisma migrate save --experimental
```

이는 `prisma/migrations` 이라는 폴더를 생성하고, 당신이 만든 수정 내역을 추적한다. 마이그레이션을 진행하자.

```
npx prisma migrate up --experimental
```

마지막으로, Prisma 클라이언트가 변경점을 확인할 수 있도록 업데이트해 주어야 한다.

```
npx prisma generate
```

![](https://images.velog.io/images/sirl/post/0d3e1894-2322-4bdf-a022-7c7c1a846c4a/prisma-flow.png)

**이제 끝났다.** 이제 song 모델에 사용 가능한 두 개의 새로운 필드가 생성되었다. 이제 데이터베이스를 업데이트한 뒤, 데이터를 넣어주도록 하자. 여기에는 여러 방법들이 있다.

1. 구식의 SQL 커맨드를 사용한다.
2. Prisma 클라이언트를 실행하는 Node.js 스크립트를 실행한다.
3. 데이터베이스 수정을 위한 시각적인 에디터를 사용한다.

나에겐 3번 방법이 가장 쉬워보인다. 운이 좋게도, Prisma는 Prisma Studio라는 멋진 도구가 있다.

```
npx prisma studio --experimental
```

![](https://images.velog.io/images/sirl/post/09bd8698-922a-4cd8-8cc5-624f3f841e9f/studio.gif)
편의를 위해서, 나는 `package.json`에 추가를 해두었다.

package.json

```
{
  "scripts": {
    "db": "prisma studio --experimental",
    "db-save": "prisma migrate save --experimental",
    "db-up": "prisma migrate up --experimental",
    "generate": "prisma generate"
  }
}
```

# Dynamic Routes

이제 우리는 노래의 유튜브 링크를 알게 되었으니, 노래에 관한 페이지를 만들어보자. Next.js를 사용하면, 대괄호를 통해 동적 라우팅을 만들 수 있다.(`pages/songs/[id].js`)

pages/songs/[id].js

```
export async function getStaticProps({ params }) {
  return {
    props: {
      song: {
        youtubeId: 'N6SQ9QoSjCI'
      }
    }
  };
}

export async function getStaticPaths() {
  return {
    paths: [
      {
        params: {
          id: '1'
        }
      }
    ],
    fallback: false
  };
}

export default ({ song }) => (
  <iframe
    width="100%"
    height="315"
    src={`https://www.youtube.com/embed/${song.youtubeId}`}
    frameBorder="0"
    allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"
    allowFullScreen
  />
);
```

`/song/4`에 가보면 데이터베이스에 값을 가져오는 것을 볼 수 있다!

# Deployment

정적 사이트 배포는 [Vercel](https://vercel.com/)보다 쉬운게 없다. Vercel CLI 설치 후, 프로젝트의 최상단에서 `vc`를 실행해라. **그럼 끝이다.**

# Conclusion

Next.js 9.3+ 와 Prisma는 참 잘 맞는다. 동적인 데이터와 정적 사이트는 결과적으로 어마어마한 유저 경험을 제공한다.

디자인이 적용된 튜토리얼 코드를 보고싶다면, [next-prisma]()https://github.com/leerob/next-prisma) 레포지토리에서 확인할 수 있다.
