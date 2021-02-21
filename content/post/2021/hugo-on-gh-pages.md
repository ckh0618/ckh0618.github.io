---
published: true
title: GH-Pages with HUGO    
date: 2021-02-19T10:42:46+09:00
last_modified_at: 2021-02-19T10:42:46+09:00
categories: [잡담]
tags: [HUGO, github pages, https]
--- 

원래 기록을 잘 하고 지식을 잘 정리하는 스타일은 아닙니다만, 어느덧 새로운 지식을 익히는 속도보다 그것을 잊는 속도가 빨라져버린 나이가 되어버려 
무언가 정리할 공간이 필요하다는 생각이 들었습니다. 그래서 예전에 처박아놓고 관리안했던 github page 를 고쳐서 사용해보기로 결심했고, 
하루 정도 시간을 투자해서 걸려서 이 사이트를 만들었습니다. 

과거에 만들어봤던 github page 는 Jeykyll 의 오랜 빌드타임과 잘 모르는 ruby, 테마에서 사용하는 javascript, 그리고 의존성, 
가끔 github 에서 보내는 보안경고등에 질려서 과감히 Jekyll 을 버리고 HUGO 를 선택하게 되었습니다. 제가 생각하는 HUGO 의 장점은
다음과 같습니다. 

* 무엇보다도 번개같은 빌드 타임 
* 바이너리하나로 대표되는 Golang 특성의 Zero Dependency 
* Golang 은 그래도 익숙 까막눈인 Ruby 보다는 나음 

# HUGO 설치 
HomeBrew 를 통해서 간단하게 설치했습니다. 

```
brew install hugo 
```

설치하고 나면 기본적인 static 사이트를 위한 skelton 을 만들 차례입니다. 적당한 빈디렉토리를 찾아서 다음과 같이 빌드합니다. 

```
hugo new site blog 
```

blog 디렉토리를 들어가면 사이트 skelton 이 만들어진걸 확인할 수 있습니다. 

# HUGO THEME 

HUGO 또한 다른 static site generator 처럼 여러가지 theme 를 제공하고 있습니다. github 에 검색하다보면 여러가지 나오는데, 
저는 hugo-tranquilpeak-theme 이게 제일 맘에 들어서 이걸 선택했습니다. 다음과 같이 theme 디렉토리에서 이걸 submodule 로 
등록해주시면됩니다. 

```
cd blog/themes
git submodule add https://github.com/kakawait/hugo-tranquilpeak-theme.git
cp hugo-tranquilpeak-theme/exampleSite/config.toml ..
```

config.toml 은 전체 사이트의 설정 파일입니다. 각각의 테마별로 제공하는 설정파일이 약간씩 다르기 때문에 예제 config.toml 파일을 
blog 디렉토리에 있는 config.toml 에 덮어씌우고 이것저것 고쳐가면서 하면 됩니다. 해당 테마에 대한 자세한 문서는 https://github.com/kakawait/hugo-tranquilpeak-theme/blob/master/docs/user.md 에 있으니 이걸 보고 세팅하시면 됩니다. 

테마를 설정하거나 여러가지 글에 쓰고 레이아웃을 정리하기 위해서는 다음과 같이 로컬에 서버를 띄우고 확인하면 좋습니다. 

```
hugo server -D
```

위와 같이 데몬을 띄우면 파일이 변경될때마다 웹사이트를 바로바로 생성해주는데, 거의 실시간에 가까울 정도로 변환이 됩니다.  실제 글이 어떻게 
쓰여지고 레이아웃들이 어떻게 쓰여지는지 바로바로 확인할 수 있어서 저는 주로 VS CODE 에서 글을 쓰고, 저장하면서 글을 씁니다. 

# 배포  
 
널리 알려진것처럼 github 에서는 github page 에서 호스팅할 수 있는데, github action 을 사용하면 글을 쓰고 github repo 에 push 하면 
알아서 site 를 만들어서 github page 에 배포하는 작업을 만들어낼 수 있습니다. 다음은 제가 사용하고 있는 github action 예제입니다. 
아 blog 디렉토리를 올릴 repo 이름은 USERNAME.github.io 이 되어야 합니다. ( USERNAME 은 github ID 입니다. )

```
name: hugo gh-pages deploy
on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps: 
    - uses: actions/checkout@v2
    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2
      with:
        hugo-version: 'latest'
        extended: true
    - name: Build Hugo Site
      run: |
        hugo
    - name: Deploy
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN  }}
        publish_branch: gh-pages
        publish_dir: ./public
        cname: mo0om.com
```

master branch 에 Push 되면 이를 감지하여 hugo 를 통해 사이트를 생성하고, 이 생성된 디렉토리 (public) 을 gh-pages 브랜치에 
push 하는 예제입니다. 


# Custom domain 연결, 그리고 https 

Custom Domain 을 github 에 직접 연결하기 위해서는 다음과 같은 절차를 거쳐야합니다. 
도메인을 설정은 약간의 시간을 필요로 할 수 있습니다. A 레코드의 변경사항은 전파되는데 시간좀 걸려요. 
저는 세팅하고 4-5시간 정도 걸렸던거 같습니다. 

1. 도메인의 A 레코드에 다음과 같은 아이피를 추가합니다. 
```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```
2. 그리고 CNAME 을 추가합니다. www 를 USERNAME.github.io 를 바라보게 세팅합니다.

3. 여러분의 repo 에서 Settings -> Github Pages 에 Enforce HTTPS 를 체크합니다. 

아마 다음과 같이 세팅되면 될겁니다. 

![설정예제](/hugo-on-gh-pages/github_page_settings.png)


이제 장소는 구비되었으니 내용만 잘 채우면 되겠군요 !
