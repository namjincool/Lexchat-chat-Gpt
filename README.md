# Amazon Lex에서 Open API를 이용한 대화형 Chatbot 구현하기

## Chatbot Architecture

여기에서 구현하는 Architecture는 아래와 같습니다. [Amazon CloudFront](https://aws.amazon.com/ko/cloudfront/)를 이용하여 채팅을 위한 웹페이지를 제공합니다. 사용자가 입력한 채팅 메시지는 [Amazon API Gateway](https://aws.amazon.com/ko/api-gateway/)와 [AWS Lambda](https://aws.amazon.com/ko/lambda/)를 이용해 Lex에서 의도(intent)를 파악후 답변을 합니다. 그런데, Lex에서 인식되지 못한 의도가 있다면 Lambda 함수를 이용하여 ChatGPT에 질의를 하고, 그 결과를 채팅창에 표시하게 됩니다. 이러한 대화형 Chatbot을 구성하기 위한 인프라는 [AWS CDK](https://aws.amazon.com/ko/cdk/)를 이용해 생성 및 관리됩니다. 모든 인프라는 [서버리스(Serverless)](https://aws.amazon.com/ko/serverless/)로 구성되므로 유지보수면에서 효율적이며 변동하는 트래픽에도 자동 확장(Auto Scaling)을 통해 안정적으로 시스템을 운용할 수 있습니다.

![image](https://user-images.githubusercontent.com/52392004/223118356-ff47ed18-de76-403c-ab88-c7583af757bf.png)

상세한 동작은 아래를 참조합니다. 

단계1: 사용자는 CloudFront의 도메인으로 Chatbot 웹페이지를 접속을 시도하여, S3에 저장된 HTML, CSS, Javascript를 로드합니다.

단계2: 웹페이지에서 채팅 메시지를 입력합니다. 이때 "/chat"리소스에 POST Method으로 JSON 포맷으로 된 text 메시지를 RESTful 형태로 요청하게 됩니다.

단계3: CloudFront는 API Gateway로 요청을 전송합니다.

단계4: API Gateway는 /chat 리소스에 연결되어 있는 Lambda 함수를 호출합니다.

단계5: Lambda 함수는 Lex V2 API를 이용하여 채팅 메시지를 Lex에 전달합니다.

단계6: Lex는 미리 정의한 의도(intent)가 있는 경우에 해당하는 동작을 수행합니다. 의도를 인식할 수 없는 메시지라면, ChatGPT로 문의하는 요청을 보냅니다.

단계7: ChatGPT에서 답변을 하면, 응답이 이전 단계의 역순으로 전달되어서 사용자에게 전달됩니다.

## 실습 과정 

### AWS CLI , CDK 설치 방법 

AWS CLI를 [설치](https://awscli.amazonaws.com/AWSCLIV2.msi)를 하고 , 이후 CDK를 사용하기 위해 Node.js를 설치해서
백엔드 환경을 구성해줍니다

```java
npm install -g aws-cdk
```

### 전체 코드 다운로드 및 CDK 배포 준비

아래와 같이 소스를 다운로드합니다.

```java
git clone https://github.com/namjincool/Lexchat-chat-Gpt.git
```

CDK 폴더로 이동하여 필요한 라이브러리를 설치합니다. 여기서 aws-cdk-lib은 CDK 2.0 라이브러리입니다.

```java
cd Lexchat-chat-Gpt/cdk-chatbot && npm install
```

CDK를 처음 사용하는 경우에는 아래와 같이 bootstrap을 실행하여야 합니다. 여기서 account-id은 12자리의 Account Number를 의미합니다. AWS 콘솔화면에서 확인하거나, "aws sts get-caller-identity --query Account --output text" 명령어로 확인할 수 있습니다.

```java
cdk bootstrap aws://account-id/ap-northeast-2
```

### Lex에서 Chatbot의 구현

[Amazon Lex 한국어 챗봇 빌드 워크숍](https://github.com/aws-samples/aws-ai-ml-workshop-kr/blob/master/aiservices/lex-korean-workshop/README.md)의 [Hello World Bot](https://github.com/aws-samples/aws-ai-ml-workshop-kr/blob/master/aiservices/lex-korean-workshop/HelloWorldBot.md)에 따라 HelloWorld Bot을 생성합니다. "Hello World Bot"은 "안녕"이라고 입력하면, 이름을 물어보고 확인하는 간단한 인사봇입니다. 

"Hello World Bot" 생성을 완료한 후에, [Bot Console](https://ap-northeast-2.console.aws.amazon.com/lexv2/home?region=ap-northeast-2#bots)에 접속해서 "HelloWorldBot"을 선택합니다. 이 후 botId와 botAliasId를 알 수 있습니다. 

### 환경변수 업데이트

Cloud9으로 돌아가서 왼쪽 파일 탐색기에서 "Lexchat-chat-Gpt/cdk-lex/lib/cdk-lex-stack.ts"을 열어서 "Lambda for lex"의 Environment의 botId, botAliasId를 업데이트 합니다. 여기서 sessionId는 현재의 값을 유지하거나 임의의 값을 입력합니다.

![noname](https://user-images.githubusercontent.com/52392004/223222609-e2dae835-66cb-4ae2-a3f8-d094c4afe6f4.png)

또한, "Lambda for chatgpt"의 environment에서 "OPENAI_API_KEY"을 입력합니다. 미리 받은 Key가 없다면 [OpenAI: API Key](https://platform.openai.com/account/api-keys)에서 발급받아서 입력합니다. 

![noname](https://user-images.githubusercontent.com/52392004/223222868-3e53dbce-fae1-4255-bde8-fd3b9d60d663.png)




### 배포하기 

이제 CDK로 전체 인프라를 생성합니다.

```java
cdk deploy
```

정상적으로 설치가 되면 아래와 같은 "Output"이 보여집니다. 여기서 distributionDomainName은 "d3ndv6lhze8yc5.cloudfront.net"이고, WebUrl은 "https://d3ndv6lhze8yc5.cloudfront.net/chat.html"임을 알 수 있습니다.

![noname](https://user-images.githubusercontent.com/52392004/222942854-065a36a8-ee7d-4a92-b7e3-9a5fbaee105d.png)



### Lex에서 Lambda 함수로 ChatGPT를 호출하도록 설정하기

[AWS Lex Console](https://ap-northeast-2.console.aws.amazon.com/lexv2/home?region=ap-northeast-2#bots)에서 "HellowWorldBot"을 선택하여 "Aliases"에서 [Languages]를 선택하여 아래처럼 [Korean(South Korea)]를 선택합니다.

![noname](https://user-images.githubusercontent.com/52392004/223216750-ceebc59e-dacc-4626-81c1-9a58f9b2a5b5.png)

아래처럼 [Souce]로 "lambda-chatgpt"를 선택하고, [Lambda function version or alias]은 "$LATEST"를 선택하고, [Save]를 선택합니다.

![noname](https://user-images.githubusercontent.com/52392004/223217113-5605aff4-f84f-4c7d-8d6b-17056460a5d8.png)

이후 "HellowWorldBot"의 [Intents]에서 아래처럼 [FallbackIntent]를 선택합니다. 

![noname](https://user-images.githubusercontent.com/52392004/223212743-056c3f3e-16b1-4590-b60e-fd30376fe2b0.png)

이후 아래로 스크롤하여 Fulfillment에서 [Advanced options]를 선택한 후, 아래 팝업의 [Use a Lambda function for fulfillment]을 Enable 합니다. 

![noname](https://user-images.githubusercontent.com/52392004/223218985-23b87c1a-4020-4996-a63f-df082743b35d.png)

화면 상단의 [Build]를 선택하여 변경된 내용을 적용합니다.

![noname](https://user-images.githubusercontent.com/52392004/223218223-c35fd42f-c75b-445c-aecf-4503aea09ce5.png)


### 실행하기 

WebUrl의 ""https://d3ndv6lhze8yc5.cloudfront.net/chat.html" 으로 브라우저에서 채팅화면으로 접속합니다. 아래와 같이 웹브라우저에서 Lex와 채팅을 할 수 있습니다. 아래의 첫 입력은 "HelloWorld" Bot에 있는 이름을 확인하는 Intent 동작입니다. 이후 나오는 질문인 "Lex에 대해 설명해줘"는 "HelloWorld" Bot에 의도(intent)로 등록되지 않은 질문이므로 ChatGPT에 문의하여 아래와 같은 결과를 사용자에게 보여줄 수 있었습니다. ChatGPT를 제공하는 OpenAI 서버의 응답속도가 지연되면, [웹 브라우저의 설정](https://stackoverflow.com/questions/39751124/increase-timeout-limit-in-google-chrome)에 따라 ChatGPT로의 응답을 일부 수신하지 못할 수 있습니다. 


## 리소스 정리하기 

더이상 인프라를 사용하지 않는 경우에 아래처럼 모든 리소스를 삭제할 수 있습니다. 

```java
cdk destroy
```

## 결론

ChatGPT가 나옴으로써 Amazon Lex를 이용한 대화형 chatbot이 사용자의 사용성을 개선하고 더 좋은 서비스를 제공할 수 있을것으로 기대됩니다.


