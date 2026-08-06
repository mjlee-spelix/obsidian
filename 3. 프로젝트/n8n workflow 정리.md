## 영업 지원

## 시장 모니터링
> 지능형 시장 모니터링 및 자동화된 뉴스레터 발행
> 시장 가격과 최신 뉴스를 스스로 판단하여 수집하고, 이를 비즈니스 보고서 양식으로 가공해 담당자에게 이메일로 전달

흐름
① 설정 단계 (Configuration)
![[Pasted image 20260311153359.png]]
- **통로:** Webhook (`configuration save on sheet`)
- **내용:** 사용자가 분석하고 싶은 키워드, 부서명, 수신 이메일 등을 입력하면 Google Sheets의 'configuration' 시트에 저장합니다.

② 조사 단계 (Research Phase) - 오전 08:50 실행
![[Pasted image 20260311153428.png]]
- **작동:** 저장된 키워드들을 루프(`Loop Over Items`)를 돌며 하나씩 처리합니다. 
- **에이전트 역할:** `Autonomous Market Research Agent`가 출동합니다.    
    - **가격 조회:** Yahoo Finance API를 통해 실시간 자산 가격을 가져옵니다.
    - **뉴스 검색:** SearXNG를 통해 최신 시장 동향을 검색합니다.    
- **결과 저장:** 수집된 분석 요약, 시장 심리(Bullish/Bearish) 등을 'search' 시트에 기록합니다.

③ 보고 단계 (Reporting Phase) - 오전 09:00 실행
![[Pasted image 20260311153444.png]]
- **작동:** 조사 단계에서 저장된 데이터를 바탕으로 보고서를 생성합니다.
- **에이전트 역할:** `Report Agent`가 출동합니다.
    - **데이터 조회:** Google Sheets에서 아까 저장된 조사 데이터를 읽어옵니다.
    - **보고서 생성:** 미리 정의된 전문 비즈니스 템플릿(개조식 문체 등)에 맞춰 한국어 보고서를 작성합니다.
- **전송:** 최종 완성된 내용을 담당자의 이메일로 발송(`Send an Email`)합니다.