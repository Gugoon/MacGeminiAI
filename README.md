# GeminiAI (macOS/iOS)

&nbsp;

# 주요 기능 및 특징 

### 사용자가 자연어로 질문하고 명령을 내릴 수 있는 채팅형 AI 앱 

## macOS Version

1. Mac OS 상단 아이콘 클릭하여 실행 가능
2. Image File을 드래그 드랍하여 업로드
3. 채팅 Message MarkDown 형식 지원
4. 채팅 히스토리 저장 및 열람

## iOS Version

1. 키보드입력 또는 음성인식(Speech To Text)으로 채팅 가능
2. 겔러리에 접근 ImageFile 업로드 (Image Picker)
3. 채팅 Message MarkDown 형식 지원
4. 채팅 히스토리 저장 및 열람
5. 흔들어서 채팅 초기화


&nbsp;

mac App 다운로드 :  [GeminiAI.app.zip](https://github.com/Gugoon/MacGeminiAI/files/14413038/GeminiAI.app.zip)

[참고] [앱 실행 이슈]


설정 -> 개인정보 보호 및 보안 -> '보안'다음에서 다운로드한 응용 프로그램 허용 -> 하단

<img width="470" alt="스크린샷 2024-02-26 10 45 18" src="https://github.com/Gugoon/MacGeminiAI/assets/10485667/0f75ac46-c1d1-4378-a154-980fb586971a">

- "그래도 열기" 선택 

&nbsp;

# macOS 실행 화면
&nbsp;

![화면 기록 2024-02-27 09 52 04](https://github.com/Gugoon/MacGeminiAI/assets/10485667/cb0fa287-6015-4871-9508-20c5d19b163c)

![화면 기록 2024-02-27 09 53 23](https://github.com/Gugoon/MacGeminiAI/assets/10485667/84ce8a23-9bea-4530-b05c-d77826be4655)

![화면 기록 2024-02-27 09 54 04](https://github.com/Gugoon/MacGeminiAI/assets/10485667/fe72eac2-ad26-447c-9824-635440ffa94c)

&nbsp;
# iOS 실행 화면
![IMG_2850](https://github.com/Gugoon/MacGeminiAI/assets/10485667/f9fff0d7-37f2-4eb7-bf53-55ef66177f97)
![IMG_2849](https://github.com/Gugoon/MacGeminiAI/assets/10485667/7037b095-1dbc-4041-90d2-454912aa80b3)
![IMG_2847](https://github.com/Gugoon/MacGeminiAI/assets/10485667/a72a1b12-4995-4852-b644-00833a520a66)
![IMG_2846](https://github.com/Gugoon/MacGeminiAI/assets/10485667/ab67a87c-d42e-4038-bd99-26430b9bee32)
![IMG_2845](https://github.com/Gugoon/MacGeminiAI/assets/10485667/bd990df5-73b0-45b4-bdfd-9a79b6dd44b9)

&nbsp;

# 주요 기술 및 코드
- SwiftUI, SwiftData, MVVM
- GoogleGenerativeAI

&nbsp;

# 코드 파일 구성

<img width="387" alt="스크린샷 2024-03-01 16 35 06" src="https://github.com/Gugoon/MacGeminiAI/assets/10485667/b4baa005-0f74-4fe7-a6f0-cc6fb34ac7b9">



```swift

//: ## SwiftData

@Model
final class HistroyItemData{
    var id : String
    @Relationship(deleteRule : .cascade, inverse: \HistoryChatData.item)
    var chatData: [HistoryChatData]? = []
    
    init(id: String) {
        self.id = id
    }
}

@Model
final class HistoryChatData{
    var id : String
    var message: String
    var participant: String
    var item : HistroyItemData?
    
    init(id: String, message: String, participant: String, item: HistroyItemData?) {
        self.id = id
        self.message = message
        self.participant = participant
        self.item = item
    }
}

```

```swift

//: ## View

struct HistoryView : View{
    //..생략
    @Query(sort: \HistroyItemData.id, order: .forward) 
        private var historyItems: [HistroyItemData]
    @StateObject var conversationViewModel: ConversationViewModel
    @StateObject var historyDataViewModel: HistoryDataViewModel
    
    @Environment(\.modelContext) private var context
    //..생략

    List(historyDataViewModel.historyItem) { historyItem in
                if let historyChatData = historyItem.chatData, let chatFirstMessage = historyChatData.first?.message {
                    HStack{
                        Text(chatFirstMessage)
                            .frame(maxWidth: .infinity, alignment: .leading)
                            .contentShape(Rectangle())
                            .onTapGesture {
                                var modelContent : [ModelContent] = []
                                for chatData in historyChatData{
                                    modelContent.append(ModelContent(role: chatData.participant == "user" ?  "user" : "model" , parts: chatData.message))
                                }
                                conversationViewModel.startRelayChat(history: modelContent)
                            }
                            .foregroundColor(.orange)
                            .lineLimit(3)
                        
                        Text("❎")
                            .onTapGesture {
                                context.delete(historyItem)
                            }
                            .padding(6)
                    }//: HSTACK
                }
                
            }//: LIST
            .background(.gray)
    //..생략            
    
    private func deleteItem(historyItem: HistroyItemData) {
        context.delete(historyItem)
        do {
            try context.save()
        } catch {
            print(error.localizedDescription)
        }
    }

    //..생략
}
```


```swift

//: ## ViewModel

import GoogleGenerativeAI
import SwiftUI

@MainActor
class ConversationViewModel: ObservableObject {
    @Published var messages = [ChatMessageData]()
    @Published var isBusy = false
    
    @Published var error: Error?
    var hasError: Bool {
        return error != nil
    }

    //..생략

    private var model: GenerativeModel
    private var chat: Chat
    private var stopGenerating = false
    
    private var chatTask: Task<Void, Never>?

    
    init() {
        model = GenerativeModel(name: "gemini-pro", apiKey: APIKey.default)
        chat = model.startChat()
    }
    

    func startNewChat() {
        stop()
        error = nil
        chat = model.startChat()
        messages.removeAll()
        ..생략    
    }

    func sendMessage(_ text: String, streaming: Bool = true, images : [NSImage]? = nil) async {
        error = nil
        
        if ((images?.isEmpty) != nil){
            await internalSendMessageWithImage(text, selectedImage: images!)
        }else{
            if streaming {
                await internalSendMessageStreaming(text)
            } else {
                await internalSendMessage(text)
            }
        }
    }

    //..생략    
    
    func stop() {
        chatTask?.cancel()
        error = nil
    }

    //..생략    

    private func internalSendMessage(_ text: String) async {
        chatTask?.cancel()
        
        chatTask = Task {
            isBusy = true
            defer {
                isBusy = false
            }
            
            let userMessage = ChatMessageData(message: text, participant: .user)
            messages.append(userMessage)
            
            let systemMessage = ChatMessageData.pending(participant: .system)
            messages.append(systemMessage)
            
            do {
                var response: GenerateContentResponse?
                response = try await chat.sendMessage(text)
                
                
                if let responseText = response?.text {
                    messages[messages.count - 1].message = responseText
                    messages[messages.count - 1].pending = false
                }
                
            } catch {
                self.error = error
                print(error.localizedDescription)
                messages.removeLast()
            }
        }
    }
    //..생략    
}

```



```swift

//: ## Model

enum Participant {
    case system
    case user
}

struct ChatMessageData: Identifiable, Equatable{
    let id = "\(Date().timeIntervalSince1970)"
    var message: String
    let participant: Participant
    var pending = false
    var selectedImage = [NSImage]()
    
    static func pending(participant: Participant) -> ChatMessageData {
        Self(message: "", participant: participant, pending: true)
    }
}

```


## 참고 자료 : https://ai.google.dev


