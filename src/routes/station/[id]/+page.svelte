<script>
  import { onMount } from 'svelte';
  import { page } from '$app/stores'; 
  import { supabase } from '$lib/supabase';
  import { timeAgo } from '$lib/index';

  // 배경 이미지 경로
  let backgroundImage = $state("/images/raindrop.jpg");

  // Svelte 5 룬 상태 관리
  let station = $state(null);
  let loading = $state(true);

  // 건의함 관련 상태
  let feedbackText = $state("");
  let feedbackSubmitted = $state(false);

  // URL 주소에서 [id] 값 추출
  const stationId = $page.params.id;

  onMount(async () => {
    // 1. 데이터 단일 조회
    async function fetchStationDetail() {
      const { data, error } = await supabase
        .from('station')
        .select('*')
        .eq('id', stationId)
        .single();

      if (error) {
        console.error('상세 데이터를 불러오는 중 오류 발생:', error);
        loading = false;
        return;
      }

      if (data) {
        station = data; 
        loading = false;
      }
    }

    await fetchStationDetail();

    // 2. 실시간 구독
    const channel = supabase
      .channel(`station-detail-${stationId}`)
      .on(
        'postgres_changes',
        {
          event: 'UPDATE',
          schema: 'public',
          table: 'station',
          filter: `id=eq.${stationId}`
        },
        (payload) => {
          console.log('현재 거치대 실시간 업데이트 감지:', payload.new);
          if (payload.new && payload.new.id === stationId) {
            station = {
              ...station,
              current_count: payload.new.current_count,
              location: payload.new.location,
              description: payload.new.description,
              image_url: payload.new.image_url,
              map_url: payload.new.map_url,
              updated_at: payload.new.updated_at
            };
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  });

  // 건의사항 제출 함수
  async function handleFeedbackSubmit(e) {
    e.preventDefault();
    if (!feedbackText.trim()) return;

    const { error } = await supabase
      .from('feedbacks') 
      .insert([
        {
          station_id: stationId,
          content: feedbackText,
        }
      ]);

    if (!error) {
      feedbackSubmitted = true;
      feedbackText = "";
      setTimeout(() => { feedbackSubmitted = false; }, 3000);
    } else {
      console.error('건의사항 제출 실패:', error);
      alert('제출에 실패했습니다. 다시 시도해 주세요.');
    }
  }
</script>

{#if loading}
<main class="glass-detail-page" style="--bg-image: url('{backgroundImage}'); --status-color: #5e5e5e">
  <a href="/" class="back-btn">⬅️ 돌아가기</a>
  <div class="detail-container" style="display: flex; justify-content: center; align-items: center; color: #fff; font-weight: bold;">
    거치대 정보를 안전하게 불러오는 중...
  </div>
</main>
{:else if station}
<main class="glass-detail-page" style="--bg-image: url('{backgroundImage}'); --status-color: {station.current_count > 0 ? '#111' : '#5e5e5e'}">
  <a href="/" class="back-btn">⬅️ 돌아가기</a>

  <div class="detail-container">
    
    <section class="left-panel glass-card">
      <div class="glass-frost-texture"></div>
      
      <div class="panel-header">
        <span class="status-tag" style="background: rgba(255,255,255,0.3); color: var(--status-color)">
          {station.current_count > 0 ? '대여 가능' : '수량 없음'}
        </span>
        <span class="status-tag" style="background: rgba(255,255,255,0.3); color: var(--status-color)">
          최근 업데이트: {timeAgo(station.updated_at)}
        </span>
      </div>

      <div class="panel-body">
        <h1 class="station-title">{station.location}</h1>
        <p class="station-desc">{station.description || '등록된 설명이 없습니다.'}</p>

        <div class="big-count-box">
          <span class="count-num">{station.current_count}</span>
          <span class="unit">개 남음</span>
        </div>
      </div>

      <div class="suggest-block">
        <h3 class="sub-title">✉️ 거치대 건의함</h3>
        <p class="suggest-info">센서 인식 불량, 우산 파손 등 불편한 점을 남겨주시면 학생회에 전달됩니다.</p>

        {#if feedbackSubmitted}
          <div class="success-banner">✔ 의견이 소중하게 접수되었습니다.</div>
        {:else}
          <form onsubmit={handleFeedbackSubmit}>
            <textarea
              bind:value={feedbackText}
              placeholder="여기에 건의사항을 적어주세요..."
              rows="3"
            ></textarea>
            <button type="submit" class="submit-btn">소중한 의견 보내기</button>
          </form>
        {/if}
      </div>
    </section>

    <section class="right-panel glass-card">
      <div class="glass-frost-texture"></div>
      <h3 class="sub-title">📍 찾아오는 길 가이드</h3>
      
      <div class="image-wrapper">
        {#if station.image_url}
          <img src={station.image_url} alt="{station.location} 실물 위치 안내" />
        {:else}
          <div class="img-placeholder">📸 거치대 가이드 이미지를 넣어주세요</div>
        {/if}
      </div>
    </section>

  </div>
</main>
{:else}
<main class="glass-detail-page" style="--bg-image: url('{backgroundImage}'); --status-color: #5e5e5e">
  <a href="/" class="back-btn">⬅️ 돌아가기</a>
  <div class="glass-card" style="text-align: center; color: #111;">
    <h2>존재하지 않는 정거장입니다.</h2>
  </div>
</main>
{/if}

<style>
  /* 1. Pretendard 폰트 등록 */
  @font-face {
    font-family: 'Pretendard';
    src: url('/fonts/PRETENDARD-MEDIUM.OTF') format('opentype');
    font-weight: 500;
    font-style: normal;
    font-display: swap;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    overflow-y: auto; /* body 스크롤 허용 */
    background-color: #111;
  }

  .glass-detail-page {
    width: 100vw;
    min-height: 100vh; /* height 대신 min-height 사용 */
    height: auto; /* 화면이 길어지면 자동으로 높이가 늘어남 */
    font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
    letter-spacing: -0.02em;
    position: relative; 
    box-sizing: border-box;
    padding: 90px 40px 40px 40px;
    display: flex; 
    justify-content: center; 
    align-items: flex-start; /* center 대신 flex-start로 변경해서 위에서부터 자연스럽게배치 */
    background-image: var(--bg-image);
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    background-attachment: fixed; /* 스크롤을 내려도 배경화면은 고정되어 훨씬 예뻐짐! */
    overflow-y: auto; /* 스크롤 허용 */
  }

  .glass-detail-page *, 
  .glass-detail-page *::before, 
  .glass-detail-page *::after {
    box-sizing: border-box;
  }

  .glass-detail-page::before {
    content: "";
    position: absolute; inset: 0;
    background: rgba(0, 0, 0, 0.3); z-index: 0;
    pointer-events: none;
  }

  .back-btn {
    position: absolute;
    top: 25px; left: 25px; z-index: 10;
    color: #111;
    text-decoration: none; font-weight: bold; font-size: 0.95rem;
    background: rgba(255, 255, 255, 0.6);
    padding: 8px 18px; border-radius: 30px;
    border: 1px solid rgba(255, 255, 255, 0.4);
    backdrop-filter: blur(10px);
    transition: background 0.2s;
  }
  .back-btn:hover { background: rgba(255, 255, 255, 0.8); }

  .detail-container {
    width: 100%;
    max-width: 1050px;
    /* max-height: calc(100vh - 140px);  <- 이 줄을 주석 처리하거나 삭제! */
    z-index: 1;
    display: grid; 
    grid-template-columns: 1fr 1fr; 
    gap: 25px;
  }

  @media (max-width: 768px) {
  .detail-container { 
    grid-template-columns: 1fr;
    /* max-height 제한 제거 */
  }
  .glass-detail-page { 
    padding-top: 85px; 
    overflow-y: auto; /* 모바일 스크롤 보장 */
  }
}

  /* 5. 공통 글래스 카드 스타일 */
  .glass-card {
    background: rgba(255, 255, 255, 0.35);
    border: 1px solid rgba(255, 255, 255, 0.4);
    border-radius: 24px;
    backdrop-filter: blur(25px) saturate(110%) !important;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.15);
    padding: 25px; position: relative; overflow: hidden;
    color: #111111;
    display: flex;
    flex-direction: column;
  }
  
  .glass-frost-texture {
    position: absolute; inset: 0;
    background-image: radial-gradient(rgba(0, 0, 0, 0.02) 1px, transparent 0);
    background-size: 4px 4px; opacity: 0.5; pointer-events: none;
  }



  .panel-header { margin-bottom: 12px; display: flex; gap: 8px; flex-wrap: wrap; }
  .status-tag { padding: 4px 12px; border-radius: 20px; font-weight: bold; font-size: 0.75rem; }
  .station-title { font-size: 1.8rem; font-weight: bold; margin: 4px 0; color: #000; }
  .station-desc { color: #333; font-size: 0.9rem; margin: 0 0 15px 0; }


  .count-num { font-size: 4rem; font-weight: normal; color: var(--status-color); line-height: 1; }
  .unit { font-size: 1.1rem; color: #111; margin-left: 8px; font-weight: bold; }

  .sub-title { font-size: 1.1rem; font-weight: bold; margin: 0 0 10px 0; color: #000; position: relative; z-index: 1; }

  /* 1. 왼쪽 패널 찢어짐 방지 */
  .left-panel {
    display: flex;
    flex-direction: column;
    justify-content: flex-start; /* space-between 대신 flex-start 사용 */
  }

  /* 2. 대형 수량 박스 여백감 있게 조정 */
  .big-count-box {
    display: flex;
    align-items: baseline; 
    justify-content: center;
    background: rgba(255, 255, 255, 0.25); 
    border-radius: 16px;
    padding: 24px 0; /* 위아래 패딩을 늘려 안정감 부여 */
    margin-top: 10px;
    margin-bottom: 25px;
  }

  /* 3. 건의함 위쪽 여백 고정 */
  .suggest-block {
    margin-top: 10px; /* 너무 멀어지지 않게 고정 */
    position: relative;
    z-index: 1;
  }
    .suggest-info { font-size: 0.8rem; color: #333; margin: 0 0 10px 0; line-height: 1.4; }
  textarea {
    width: 100%;
    box-sizing: border-box; background: rgba(255, 255, 255, 0.4);
    border: 1px solid rgba(0, 0, 0, 0.15); border-radius: 12px; padding: 12px;
    font-family: inherit; font-size: 0.85rem; color: #000; resize: none;
  }
  textarea:focus { outline: none; border-color: var(--status-color); background: rgba(255, 255, 255, 0.6); }

  .submit-btn {
    width: 100%;
    background: #111; color: #fff; border: none; padding: 12px;
    border-radius: 12px;
    font-weight: bold; font-size: 0.9rem; cursor: pointer;
    margin-top: 8px; transition: background 0.2s;
  }
  .submit-btn:hover { background: #333; }

  .success-banner {
    background: rgba(255, 255, 255, 0.4);
    border: 1px solid var(--status-color);
    color: var(--status-color); padding: 12px;
    text-align: center; font-weight: bold; border-radius: 12px; font-size: 0.85rem;
  }

  /* 오른쪽 패널 (사진 전용 카드) */
  .right-panel {
    display: flex;
    flex-direction: column;
  }

  .image-wrapper { 
    width: 100%;
    flex: 1; /* 사진 영역이 남아있는 하단 높이를 가득 채움 */
    min-height: 250px;
    overflow: hidden; 
    border-radius: 16px;
    border: 1px solid rgba(0, 0, 0, 0.1); 
    position: relative;
    z-index: 1;
  }
  
  img { 
    width: 100%;
    height: 100%;
    object-fit: cover; 
    display: block;
  }
  
  .img-placeholder {
    height: 100%; background: rgba(0, 0, 0, 0.05); color: #444;
    font-size: 0.85rem; display: flex; align-items: center; justify-content: center; font-weight: bold;
  }
</style>