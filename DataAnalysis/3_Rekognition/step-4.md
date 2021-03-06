## 4 단계 : 레이블 및 이미지 메타 데이터 생성

이제 상태 머신을 통해 이미지의 섬네일과 메타 데이터 생성뿐 아니라 이미지 분석(태깅)이 가능하게 되었습니다.


### 4A 단계 : DynamoDB에 메타 데이터를 저장하는 단계 추가

이 부분에서는 다시 JSON을 수동으로 편집할 것입니다. 이전에 3 단계에서 작성한 JSON으로 이동하여 추가합니다. 여기에서 DynamoDB에 모든 것을 저장하는 람다 함수 `sfn-workshop-setup-PersistDDB`를 실행하기 위해 상태 머신에 한 단계 더 추가하십시오.

구문에 대한 도움이 필요한 경우 이 설명서를 살펴보세요.

- [작업 상태](https://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-task-state.html)
- [Amazon States Language Spec](https://states-language.net/spec.html)

1. 3 단계에서 작성한 JSON에서 Parallel 다음 이어서 작성된 JSON을 복사, 붙여넣기 하세요. 문제가 발생하면 이 페이지 하단에서 JSON을 찾으세요.

	```JSON
        ,
        "PersistDDB": {
          "Type": "Task",
          "Resource": "arn:aws:lambda:ap-northeast-2:012345678901:function:sfn-workshop-setup-PersistDDB",
          "End": true
        }
	```

	> 람다 함수의 ARN을 찾으려면 AWS CloudFormation 콘솔에서 **sfn-workshop-setup** 스택으로 이동하여 출력 섹션에서 **StoreImageMetadataFunction**을 찾습니다.
	>
	>`arn:aws:lambda:ap-northeast-2:<YOUR-ACCOUNT-ID>:function:sfn-workshop-setup-PersistDDB`와 같이 보일 것입니다.

1. **Visual Workflow**옆의 &#x21ba; 아이콘을 클릭하여 상태 머신의 시각적 표현을 새로 고칩니다.

	<img src="images/4b-step-console-preview.png" width="90%">

1. **업데이트**를 클릭하세요.


### 4B 단계 : 상태 머신 실행 테스트

1. 지금까지 사용해온 다음 입력을 사용하여 실행을 테스트하세요.

	```JSON
	{
	  "s3Bucket": "FILL_IN_YOUR_VALUE",
	  "s3Key": "tests/1_cactus.jpg"
	}
	```

1. 모든 것이 정상적으로 작동하고 워크 플로가 모든 단계를 성공적으로 수행했는지 확인합니다.

 	<img src="images/4c-test-choice-test-succeeded.png" width="50%">

1. DynamoDB 테이블에 저장된 항목이 어떤 것인지 확인할 수 있습니다. [Amazon DynamoDB 관리 콘솔](https://console.aws.amazon.com/dynamodb/home?#tables:)로 이동하세요. 다음 테이블과 비슷한 이름의 테이블이 두 개 있어야합니다.

    - `sfn-workshop-setup-AlbumMetadataDDBTable`
 	- `sfn-workshop-setup-ImageMetadataDDBTable`

1. 'sfn-workshop-setup-ImageMetadataDDBTable`을 클릭하고 **항목** 탭를 클릭하세요. 성공적으로 처리된 이미지에 대해 항목이 있는지 확인하세요.

 	<img src="images/4c-ddb-imagemetadata.png" width="90%">


1. 항목을 클릭하고 데이터 구조를 살펴보세요.

 	<img src="images/4c-ddb-imagemetadata-item.png" width="90%">


### 최종 JSON
<details>
<summary><strong> JSON 정의를 확장합니다.</strong></summary><p>

```JSON
{
  "StartAt": "ExtractImageMetadata",
  "Comment": "Image Processing State Machine",
  "States": {
    "ExtractImageMetadata": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:012345678901:function:sfn-workshop-setup-ExtractMetadata",
      "Catch": [
        {
          "ErrorEquals": [
            "ImageIdentifyError"
          ],
          "ResultPath": "$.error",
          "Next": "NotSupportedImageType"
        }
      ],
      "ResultPath": "$.extractedMetadata",
      "Next": "ImageTypeCheck"
    },
    "NotSupportedImageType": {
      "Type": "Fail",
      "Cause": "Image type not supported!",
      "Error": "FileNotSupported"
    },
    "ImageTypeCheck": {
      "Type": "Choice",
      "Choices": [
        {
          "Or": [
            {
              "Variable": "$.extractedMetadata.format",
              "StringEquals": "JPEG"
            },
            {
              "Variable": "$.extractedMetadata.format",
              "StringEquals": "PNG"
            }
          ],
          "Next": "Parallel"
        }
      ],
      "Default": "NotSupportedImageType"
    },
    "Parallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "DetectLabelsRekognition",
          "States": {
            "DetectLabelsRekognition": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:ap-northeast-2:012345678901:function:sfn-workshop-setup-DetectLabel",
              "End": true
            }
          }
        },
        {
          "StartAt": "Thumbnail",
          "States": {
            "Thumbnail": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:ap-northeast-2:012345678901:function:sfn-workshop-setup-Thumbnail",
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.parallelResults",
      "Next": "PersistDDB"
    },
    "PersistDDB": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:012345678901:function:sfn-workshop-setup-PersistDDB",
      "End": true
    }
  }
}
```
</details>

### 다음 단계
이제 [5 단계](step-5.md)로 이동할 준비가 되었습니다!
