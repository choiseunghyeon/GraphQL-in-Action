# GraphQL 작업 수정 및 구성

- 필드는 함수와 같다.
  - 함수는 입력과 출력을 매핑하며, 입력은 여러개의 인수를 통해 받는다.
  - GraphQL 필드에도 여러 인숫값을 전달할 수 있다.
- 단일 레코드를 요구하는 경우 레코드용 식별자를 지정해야 한다.

```GraphQL
query UserInfo {
    # 필드 인수 부분(email)
    user(email: "jane@doe.name") {
        firstName
        lastName
        username
    }
}
```

- 어떤 GraphQL API는 시스템상에 존재하는 모든 객체를 단일 레코드 필드로 관리한다.  
  이것을 노드 인터페이스라고 한다.(Relay에 의해 많이 알려짐)  
  시스템 전체에서 고유한 전역 ID를 통해 데이터 그래프에 있는 어떤 데이터든 찾을 수 있다.  
  찾은 후에는 인라인 조각(inline-fragment)를 사용해서 노드의 종류에 따라 확인하고 싶은 속성을 지정하면 된다.

```GraphQL
query NodeInfo {
    node(id: "A-GLOBALLY-UNIQUE-ID-HERE") {
        ... on USER {
            firstName
            lastName
            username
            email
        }
    }
}
```

- 리스트 필드가 반환하는 레코드 수 제한 및 정렬

```GraphQL
query OrgPopularRepos {
  organization(login: "jscomplete") {
    name
    description
    repositories (first: 10, orderBy: {field: STARGAZERS, direction: ASC}) {
      nodes {
        name
      }
    }
  }
}
```

- 레코드 리스트의 페이지 매김  
  pagination 기능에서는 페이지당 레코드 수를 지정해야 한다.  
  github API에서는 first인수는 after를 last인수는 before를 사용해서 구현할 수 있다.  
  검색 결과 마지막 cursor값을 after 값으로 넣어주면 그 다음 번째 리스트를 가져올 수 있다.

```GraphQL
# 1페이지
query OrgRepoConnection {
  organization(login: "jscomplete") {
    repositories(first: 10, orderBy: { field: CREATED_AT, direction: ASC}) {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

```GraphQL
# 2페이지
query OrgRepoConnection {
  organization(login: "jscomplete") {
    repositories(first: 10, after: "Y3Vyc29yOnYyOpK5MjAxNy0wMS0yMlQwMTo1NTo0MyswOTowMM4Ev4A3", orderBy: { field: CREATED_AT, direction: ASC}) {
      edges {
        cursor
        node {
          name
        }
      }
    }
  }
}
```

- 검색 및 필터링
  필드 인수를 사용하면 필터링 조건이나 검색어를 지정해서 리스트가 반환하는 결과의 범위를 좁힐 수 있다.

```GraphQL
query Search {
  repository(owner: "twbs", name: "bootstrap") {
    projects(search: "v4.1", first: 10) {
      nodes {
        name
      }
    }
  }
}
```

- 별칭

```GraphQL
query ProfileInfo {
  user(login: "samerbuna") {
    name
    company
    bio
  }
}
```

```GraphQL
# 요청
query ProfileInfo {
  user(login: "samerbuna") {
    name
    # 별칭
    companyName: company
    bio
  }
}
```

```GraphQL
# 응답
{
  "data": {
    "user": {
      "name": "Samer Buna",
      "companyName": null,
      "bio": ""
    }
  }
}
```

- 지시문을 사용한 응답 변경  
  응답 데이터의 일부를 조건에 따라 포함하거나 제외할 때 사용  
  필드뿐만 아니라 조각과 최상위 작업에도 사용이 가능하다.  
  @로 시작하는 문자열을 사용해 작성한다.  
  @include, @skip, @deprecated 3가지가 기본 지시문이다.  
  특정 스키마는 더 많은 지시문을 사용할 수 있다.

```GraphQL
# 스키마가 지원하는 모든 지시문 확인
# 지시문이 사용되는 위치, 사용할 수 있는 모든 인수 리스트를 확인할 수 있다.
query AllDirectives {
  __schema {
    directives {
      name
      description
      locations
      args {
        name
        description
        defaultValue
      }
    }
  }
}
```

- 변수와 입력값  
  \$기호로 시작하며 \$login과 같은 형태이다.  
  $기호 뒤에 지정하는 이름은 어떤 것이든 상관없다.  
  GraphQL 작업을 재사용하거나 값이 문자열의 하드코딩을 방지하기 위해 변수를 사용한다.  
  필드나 지시문에도 사용해서 입력값을 받게 만들 수도 있다.

```GraphQL
# 요청
# $orgLogin의 타입은 인수가 사용될 곳의 타입과 일치해야 한다.
query OrgInfo($orgLogin: String!){
  organization(login: $orgLogin) {
    name
    description
    websiteUrl
  }
}
```

```
# Variables
{
  "orgLogin": "jscomplete"
}
```

```GraphQL
# 요청
# 기본값 사용
# ! non-nullable 기호 지정하지 않아도됌
query OrgInfo($orgLogin: String = "jscomplete"){
  organization(login: $orgLogin) {
    name
    description
    websiteUrl
  }
}
```

- @include 지시문  
  필드 또는 조각 뒤에 조건(if 인수)을 지정하여 응답에 포함시킬지 정하기 위해 사용한다.

```
# 사용방법
# $someTest가 true면 필드 포함 false면 필드 제외
fieldName @include(if: $someTest)
```

```GraphQL
# 요청
query OrgInfo($orgLogin: String!, $fullDetails:Boolean!) {
  organization(login: $orgLogin) {
    name
    description
    websiteUrl @include(if: $fullDetails)
  }
}
```

```GraphQL
# Variables
{
  "orgLogin": "jscomplete",
  "fullDetails": false
}
```

- @skip 지시문  
  @include와 반대되는 개념이다.  
  필드 또는 조각 뒤에 조건(if 인수)을 지정하여 응답에서 제외시킬지 정하기 위해 사용한다.  
  변수 이름을 부정적으로 선언했을 때 유용하다.

```
# 사용방법
# $someTest가 true면 필드 제외 false면 필드 포함
fieldName @skip(if: $someTest)
```

```GraphQL
# 요청
query OrgInfo($orgLogin: String!, $partialDetails:Boolean!) {
  organization(login: $orgLogin) {
    name
    description
    websiteUrl @skip(if: $partialDetails)
  }
}
```

```GraphQL
# Variables
{
  "orgLogin": "jscomplete",
  "partialDetails": true
}
```

@inlcude와 @skip 같이 사용하기  
서로 간에 우선순위는 없다.  
이 경우에서는 @include가 true이고 @skip이 false일 때만 해당 필드가 포함된다.

```GraphQL
# 요청
query OrgInfo($orgLogin: String!, $partialDetails:Boolean!) {
  organization(login: $orgLogin) {
    name
    description
    websiteUrl @skip(if: $partialDetails) @include(if: true)
  }
}
```

```GraphQL
# Variables
{
  "orgLogin": "jscomplete",
  "partialDetails": true
}
```

- @deprecated 지시문  
  폐지된 스키마를 사용자에게 알려주는 용도  
  ex. 특정 타입의 필드나 ENUM값 등이 폐지되는 경우

```GraphQL
type User {
  emailAddress: String
  email: String @deprecated(reason: "Use 'emailAddress'.")
}
```

- Fragment  
  복잡하다면 작은 부분으로 나누어 처리하는게 좋은 접근법이다.  
  부분들은 가능하면 서로 의존하지 않게 독립적으로 설계해야 각각 테스트 및 재사용이 가능하다.  
  GraphQL에서 조각이란 언어의 조합 단위로, 큰 GraphQL 작업을 작은 부분으로 나눌 수 있게 해준다.  
  즉, Fragment는 GraphQL을 작은 부품으로 나누어 재사용할 수 있게 만든다.  
  또한, 필드 그룹의 중복 방지를 위해서도 사용된다.

- Fragment 정의하고 사용하기  
  조각은 선택 세트이므로 객체 타입에만 정의할 수 있다.  
  즉, 조각은 스칼라값으로 정의할 수는 없다.

```GraphQL
query OrgInfo {
  organization(login: "jscomplete") {
    name
    description
    websiteUrl
  }
}
```

```GraphQL
# Fragment로 분리
# on Organization을 타입 조건(type condition)이라고 한다.
fragment orgFields on Organization {
  name
  description
  websiteUrl
}
```

```GraphQL
# fragment 사용
query OrgInfo {
  organization(login: "jscomplete") {
    # 조각 전개(fragment spread)
    # 일반 필드를 사용한 곳이라면 어디든지 적용할 수 있다.
    # 조각의 타입이 해당 조각을 사용하고자 하는 객체의 타입과 일치할 때만 사용할 수 있다.
    # Fragment를 정의했다면 반드시 사용해야 한다.
    ...orgFields
  }
}
```

- 조각과 드라이(DRY)

```GraphQL
# 반복되는 부분을 가진 쿼리
query MyRepos {
  viewer {
    ownedRepos: repositories(affiliations: OWNER, first: 10) {
      nodes {
        nameWithOwner
        description
        forkCount
      }
    }
    orgsRepos: repositories(affiliations: ORGANIZATION_MEMBER, first: 10) {
      nodes {
        nameWithOwner
        description
        forkCount
      }
    }
  }
}
```

```GraphQL
# 중복 최소화한 쿼리
query MyRepos {
  viewer {
    ownedRepos: repositories(affiliations: OWNER, first: 10) {
      ...repoInfo
    }
    orgsRepos: repositories(affiliations: ORGANIZATION_MEMBER, first: 10) {
      ...repoInfo
    }
  }
}

fragment repoInfo on RepositoryConnection {
  nodes {
    nameWithOwner
    description
    forkCount
  }
}
```

- 조각과 UI 컴포넌트

모든 컴포넌트는 특정 데이터에 의존하고 있다.  
애플리케이션이 필요로 하는 데이터는 애플리케이션의 개별 컴포넌트가 필요로 하는 모든 데이터를 조합한 것으로, GrapQL 조각은 작은 단위의 쿼리를 조합해서 하나의 큰 쿼리를 만들 수 있게 해준다.  
따라서 GraphQL 조각이 완벽하게 컴포넌트와 연계되는 이유이다.

트위터 예제에서 헤더, 사이드바, 트윗리스트, 트윗만 사용해서 프로필 페이지를 만들어보자.

```GraphQL
fragment headerData on User {
  tweetsCount
  profileImageUrl
  backgroundImageUrl
  name
  handle
  bio
  location
  url
}
```

```GraphQL
fragment sidebarData on User {
  SuggestedFollowind {
    profileImageUrl
  }
  media {
    mediaUrl
  }
}
```

```GraphQL
fragment tweetData on Tweet {
 user {
  name
  handle
 }
 createdAt
 body
}
```

```GraphQL
fragment tweetListData on TweetList {
 tweets: {
  ...tweetData
 }
}
```

전체 페이지가 필요로 하는 데이터를 만들려면 조각 전개를 사용해서 하나로 합친 쿼리를 만들면 된다.  
데이터를 받은 후에는 어떤 컴포넌트가 응답의 어떤 부분을 요청했는지 확인해서 해당 부분만 사용하도록 만들 수 있다.  
이를 통해, 컴포넌트가 필요로 하지 않는 데이터는 제외할 수 있다.

조각의 이점은, 모든 컴포넌트가 필요한 데이터를 자율적으로 선언한다.  
부모 컴포넌트에 의지하지 않고 필요한 경우 데이터 요건을 자유롭게 변경할 수 있다.

모든 UI 컴포넌트를 GraphQL 조각과 연동시키므로 각 컴포넌트에게 독립성을 부여할 수 있다.

```GraphQL
query ProfilePageData {
  user(handle: "ManningBooks") {
    ...headerData
    ...sidebarData
    ...tweetListData
  }
}
```

- 인터페이스와 유니온용 인라인 조각  
  인라인 조각은 익명 함수와 어떤 면에서 비슷하다.  
  인라인 조각은 이름이 없는 조각으로 정의한 위치에 바로 인라인으로 전개할 수 있다.  
  인터페이스나 유니온을 요청할 때 타입 조건으로 사용할 수 있다.

```GraphQL
query InlineFragment {
  repository(owner: "facebook", name: "graphql") {
    ref(qualifiedName: "master") {
      target {
        ... on Commit {
          message
        }
      }
    }
  }
}
```

인터페이스와 유니온은 추상 타입이다.  
인터페이스는 공유 필드의 리스트를 정의하고 유니온은 사용할 수 있는 객체 타입의 리스트를 정의한다.  
객체 타입은 인터페이스를 구현할 수 있으며 객체 타입이 구현한 해당 인터페이스가 정의한 필드 리스트도 함께 구현된다.  
객체 타입을 유니온으로 정의하면 객체가 반환하는 것이 해당 유니온 중 하나라는 것을 보장한다.

유니온은 기본적으로 OR 로직과 같다.

```GraphQL
# 요청
query TestSearch {
  search(first: 10, query: "graphql", type: USER) {
    nodes {
      ... on User {
        name
        bio
      }

      ... on Organization {
        login
        description
      }
    }
  }
}
```

```GraphQL
# 응답
{
  "data": {
    "search": {
      "nodes": [
        {
          "login": "graphql"
        },
        {
          "login": "graphql-python"
        },
        {
          "login": "apollographql"
        },
        {
          "login": "graphql-java"
        },
        {
          "login": "graphql-java-kickstart"
        },
        {
          "login": "graphql-dotnet"
        },
        {
          "name": "Ivan Goncharov"
        },
        {
          "name": "Masahiro Wakame"
        },
        {
          "login": "graphql-editor"
        },
        {
          "login": "graphqlwtf"
        }
      ]
    }
  }
}
```

# 정리

요청을 보낼 때 필드에 인수를 지정할 수 있다.

- 단일 레코드를 찾기, 필터, 검색, 페이지네이션 등에 활용
- 변경을 위한 입력값 제공으로도 활용

필드에 별칭을 사용할 수 있다.

지시문은 응답 구조를 애플리케이션이 선별적으로 변경할 수 있다.

지시문과 필드 인수는 요청 변수와 같이 사용할 수 있다.

Fragment를 사용해서 Query의 공통 부분을 재사용할 수 있다.  
여러 Fragment를 합쳐서 전체 쿼리를 구성할 수도 있다.  
UI 컴포넌트와 데이터를 연동시킬 때 유용한 접근법이다.

인라인 조각을 사용하면 유니온 객체 타입이나 인터페이스를 구현한 객체 타입으로부터 정보를 선별해서 취할 수 있다.
