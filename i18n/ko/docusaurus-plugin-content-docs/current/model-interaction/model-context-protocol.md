---
sidebar_position: 5
description: 모델 컨텍스트 프로토콜(MCP)이 무엇이며 어떻게 활용되는지 알아봅니다.
keywords:
    - Model Context Protocol, MCP protocol
    - MCP host, MCP clients, MCP servers
---

# 모델 컨텍스트 프로토콜(Model Context Protocol)

[모델 컨텍스트 프로토콜(Model Context Protocol, MCP)](https://modelcontextprotocol.io/introduction)은
AI 어시스턴트가 다양한 외부 서비스와 데이터 소스에 연결해 이를 사용할 수 있게 해
주는 오픈소스 통신 표준입니다.

MCP는 다음 구성 요소로 이루어진 클라이언트-서버 아키텍처를 사용합니다.

- **MCP 호스트**: AI 어시스턴트가 위치하는 곳입니다. Claude Desktop 같은 채팅
  애플리케이션, IDE 코드 어시스턴트, 또는 그 밖의 AI 기반 애플리케이션이 될 수
  있습니다. 호스트는 하나 이상의 MCP 클라이언트를 포함할 수 있습니다.
- **MCP 클라이언트**: 클라이언트는 호스트 내부의 저수준 구현으로, MCP 서버와
  일대일 연결을 유지합니다.
- **MCP 서버**: 다양한 기능과 데이터 소스를 노출하는 커넥터입니다. 각 서버는
  데이터베이스, 서드파티 API, GitHub 저장소, 로컬 파일 등 여러 백엔드나 그 밖의
  데이터 소스에 연결할 수 있습니다. 여러 서버가 로컬 머신에서 동시에 실행되거나
  원격 서비스에 연결될 수 있습니다.
- **MCP 프로토콜**: 연결된 서버 수와 관계없이 호스트와 서버 간 통신을 가능하게
  하는 전송 계층입니다.

<figure>
  <Diagram name="mcp-architecture" alt="" />
  <figcaption><b>그림 1.</b> MCP 호스트, 클라이언트, 서버, 백엔드.</figcaption>
</figure>

AI 어시스턴트가 외부 데이터나 도구에 접근해야 할 때 일어나는 일을 큰 흐름으로
보면 다음과 같습니다.

1. MCP 호스트가 MCP 프로토콜을 통해 요청을 보냅니다.
2. 알맞은 MCP 서버가 요청을 받습니다.
3. 서버가 실제 데이터 소스(데이터베이스, API, 파일 시스템 등)에 연결합니다.
4. 서버가 요청을 처리하고 프로토콜을 통해 데이터를 반환합니다.
5. AI 어시스턴트가 정보를 받아 응답에 사용할 수 있습니다.
