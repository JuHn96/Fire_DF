    <!-- SECTION 7 -->
    <div class="section">
      <h2>7. Docker 컨테이너들끼리 연결·묶이는 원리와 방법</h2>

      <h3>1) fire_net 네트워크로 같은 “가상 스위치”에 묶는다</h3>
      <p>docker-compose.yml 맨 아래에 정의한 다음 부분이 핵심입니다.</p>

      <pre><code>networks:
  fire_net:
    driver: bridge</code></pre>

      <ul>
        <li><strong>bridge</strong> 드라이버는 도커가 만드는 가상 스위치 역할입니다.</li>
        <li>모든 서비스에 <code>networks: - fire_net</code> 를 지정하면, 전부 이 스위치에 꽂힌 상태가 됩니다.</li>
        <li>같은 네트워크 안에서는 서로를 서비스 이름으로 찾을 수 있습니다.</li>
      </ul>

      <h3>2) 서비스 이름이 곧 “호스트 이름(DNS 이름)”이 된다</h3>
      <p>예를 들어, backend에서 DB에 접속할 때는 이렇게 씁니다.</p>

      <pre><code>DB_HOST = "db"
DB_PORT = 3306</code></pre>

      <p>혹은 AI 서버에서 백엔드로 요청 보낼 때:</p>

      <pre><code>BACKEND_API_URL = "http://backend:8000"</code></pre>

      <ul>
        <li><code>db</code>, <code>backend</code> 는 도커 네트워크 내부 DNS 이름입니다.</li>
        <li>컨테이너들은 이 이름으로 서로를 찾고, 실제 IP는 도커가 내부적으로 매핑합니다.</li>
        <li>그래서 IP를 직접 적을 필요가 없고, 재시작으로 IP가 바뀌어도 문제 없이 동작합니다.</li>
      </ul>

      <h3>3) depends_on 으로 “기동 순서”를 맞춘다</h3>
      <p>각 서비스에 있는 <code>depends_on</code> 설정은...</p>

      <pre><code>backend depends_on db
ai-server depends_on backend
frontend depends_on backend</code></pre>

      <ul>
        <li>먼저 <code>db</code>가 올라오고, 그 다음 <code>backend</code>, 그 다음 <code>ai-server</code>, 마지막으로 <code>frontend</code> 가 올라옵니다.</li>
        <li>이렇게 하지 않으면, AI가 먼저 떠서 아직 준비 안 된 backend에 요청을 보내다가 에러가 날 수 있습니다.</li>
      </ul>

      <h3>4) 내부 통신에는 내부 포트만, 외부 공개는 필요한 포트만</h3>

      <pre><code>backend 내부 포트: 8000
db 내부 포트: 3306</code></pre>

      <p>외부(브라우저, 워크벤치 등)에서 접근해야 하는 경우에만 <code>ports</code>로 매핑합니다.</p>

      <pre><code>db: "3307:3306"
frontend: "8080:80"</code></pre>

      <ul>
        <li>운영 환경에서는 frontend(8080 또는 80/443)만 열고 DB/BE 포트는 외부로 안 열어도 됩니다.</li>
      </ul>

      <h3>5) 컨테이너가 죽어도 전체 구조는 유지된다</h3>
      <ul>
        <li>각 서비스는 <code>restart: unless-stopped</code> 덕분에 자동 재시작됩니다.</li>
        <li>DB 데이터는 <code>mysql_data</code> 볼륨에 저장되어 컨테이너 삭제 후에도 그대로 남습니다.</li>
        <li>네트워크는 fire_net 하나로 묶여 있어서, 재시작해도 서비스 이름은 변하지 않습니다.</li>
      </ul>

      <p>결과적으로, 이 docker-compose 구성은:</p>

      <ul>
        <li>같은 네트워크(fire_net)로 컨테이너들을 한 묶음으로 만들고,</li>
        <li>서비스 이름으로 서로를 찾게 하고,</li>
        <li>depends_on으로 실행 순서를 맞추고,</li>
        <li>볼륨으로 데이터는 따로 안전하게 보존하는 구조입니다.</li>
      </ul>

    </div>