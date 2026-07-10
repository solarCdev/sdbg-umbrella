<script>
  import { onMount } from 'svelte';
  import { page } from '$app/stores'; 
  import { supabase } from '$lib/supabase'; // 네 lib 폴더 안의 supabase 설정 파일 경로에 맞춤

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

    // 2. 🔥 실시간 구독 (클로저 버그 방지 및 안전한 오브젝트 재할당)
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
              map_url: payload.new.map_url
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
      </div>

      <div class="panel-body">
        <h1 class="station-title">{station.location}</h1>
        <p class="station-desc">{station.description || '등록된 설명이 없습니다.'}</p>

        <div class="big-count-box">
          <span class="count-num">{station.current_count}</span>
          <span class="unit">개 남음</span>
        </div>

        <div class="content-block">
          <h3 class="sub-title">📍 찾아오는 길 가이드</h3>
          <div class="image-wrapper">
            {#if station.image_url}
              <img src={station.image_url} alt="{station.location} 실물 위치 안내" />
            {:else}
              <div class="img-placeholder">📸 거치대 가이드 이미지를 넣어주세요</div>
            {/if}
          </div>
        </div>
      </div>
    </section>

    <section class="right-panel">
      <div class="map-block glass-card">
        <div class="glass-frost-texture"></div>
        <h3 class="sub-title">🗺️ 지도 내 위치</h3>
        <div class="image-wrapper">
          {#if station.map_url}
            <img src={station.map_url} alt="학교 지도 내 위치 표시" />
          {:else}
            <div class="img-placeholder">🗺️ 학교 지도 캡처 이미지를 넣어주세요</div>
          {/if}
        </div>
      </div>

      <div class="suggest-block glass-card">
        <div class="glass-frost-texture"></div>
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

  /* 2. 브라우저 기본 마진 제거 및 스크롤바 원천 차단 */
  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden; 
    background-color: #111;
  }

  /* 3. 전체 레이아웃 (100vh 고정 및 정중앙 정렬) */
  .glass-detail-page {
    width: 100vw;
    height: 100vh; 
    font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
    letter-spacing: -0.02em;
    position: relative; 
    box-sizing: border-box;
    padding: 90px 40px 40px 40px;
    display: flex; 
    justify-content: center; 
    align-items: center;
    background-image: var(--bg-image);
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
  }

  .glass-detail-page *, 
  .glass-detail-page *::before, 
  .glass-detail-page *::after {
    box-sizing: border-box;
  }

  .glass-detail-page::before {
    content: "";
    position: absolute; inset: 0;
    background: rgba(0, 0, 0, 0.3); z-index: 0; pointer-events: none;
  }

  .back-btn {
    position: absolute;
    top: 25px; left: 25px; z-index: 10;
    color: #111; text-decoration: none; font-weight: bold; font-size: 0.95rem;
    background: rgba(255, 255, 255, 0.6);
    padding: 8px 18px; border-radius: 30px;
    border: 1px solid rgba(255, 255, 255, 0.4);
    -webkit-backdrop-filter: blur(10px);
    backdrop-filter: blur(10px);
    transition: background 0.2s;
  }
  .back-btn:hover { background: rgba(255, 255, 255, 0.8); }

  .detail-container {
    width: 100%;
    max-width: 1000px; 
    max-height: calc(100vh - 150px); 
    z-index: 1;
    display: grid; 
    grid-template-columns: 1.1fr 0.9fr; 
    gap: 25px;
  }

  @media (max-width: 768px) {
    .detail-container { 
      grid-template-columns: 1fr;
      max-height: calc(100vh - 120px);
      overflow-y: auto; 
    }
    .glass-detail-page { padding-top: 85px; overflow: hidden; }
  }

  .glass-card {
    background: rgba(255, 255, 255, 0.35);
    border: 1px solid rgba(255, 255, 255, 0.4);
    border-radius: 24px;
    backdrop-filter: blur(25px) saturate(110%) !important;
    /* -webkit-backdrop-filter: blur(25px) saturate(110%) !important; */
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.15);
    padding: 25px; position: relative; overflow: hidden;
    color: #111111;
  }
  
  .glass-frost-texture {
    position: absolute; inset: 0;
    background-image: radial-gradient(rgba(0, 0, 0, 0.02) 1px, transparent 0);
    background-size: 4px 4px; opacity: 0.5; pointer-events: none;
  }

  .panel-header { margin-bottom: 15px; }
  .status-tag { padding: 4px 12px; border-radius: 20px; font-weight: bold; font-size: 0.75rem; }
  .station-title { font-size: 1.8rem; font-weight: bold; margin: 0 0 8px 0; color: #000; }
  .station-desc { color: #333; font-size: 0.95rem; margin: 0 0 20px 0; }

  .big-count-box {
    display: flex;
    align-items: baseline; justify-content: center;
    background: rgba(255, 255, 255, 0.25); border-radius: 16px; padding: 15px 0; margin-bottom: 20px;
  }
  .count-num { font-size: 5rem; font-weight: normal; color: var(--status-color); line-height: 1; }
  .unit { font-size: 1.2rem; color: #111; margin-left: 8px; font-weight: bold; }

  .sub-title { font-size: 1.1rem; font-weight: bold; margin: 0 0 12px 0; color: #000; }
  .content-block { margin-top: 15px; }

  .image-wrapper { 
    width: 100%;
    max-height: 180px; 
    overflow: hidden; 
    border-radius: 14px;
    border: 1px solid rgba(0, 0, 0, 0.1); 
  }
  img { 
    width: 100%;
    height: 100%;
    object-fit: cover; 
    display: block;
  }
  .img-placeholder {
    height: 140px; background: rgba(0, 0, 0, 0.05); color: #444;
    font-size: 0.85rem; display: flex; align-items: center; justify-content: center; font-weight: bold;
  }

  .right-panel { display: flex; flex-direction: column; gap: 25px; }

  .suggest-info { font-size: 0.85rem; color: #333; margin: 0 0 12px 0; line-height: 1.4; }
  textarea {
    width: 100%;
    box-sizing: border-box; background: rgba(255, 255, 255, 0.4);
    border: 1px solid rgba(0, 0, 0, 0.15); border-radius: 12px; padding: 12px;
    font-family: inherit; font-size: 0.9rem; color: #000; resize: none;
  }
  textarea:focus { outline: none; border-color: var(--status-color); background: rgba(255, 255, 255, 0.6); }

  .submit-btn {
    width: 100%;
    background: #111; color: #fff; border: none; padding: 12px;
    border-radius: 12px; font-weight: bold; font-size: 0.9rem; cursor: pointer;
    margin-top: 10px; transition: background 0.2s;
  }
  .submit-btn:hover { background: #333; }

  .success-banner {
    background: rgba(255, 255, 255, 0.4);
    border: 1px solid var(--status-color);
    color: var(--status-color); padding: 14px; text-align: center; font-weight: bold; border-radius: 12px;
  }
</style>