<script>
	import { onMount } from 'svelte';

    import { supabase } from '$lib/supabase'; // 방금 만든 클라이언트 불러오기

	let umbrellas = $state([]); // Svelte 5 룬 상태 변수
	let lastScrollX = 0;

	onMount(() => {
    // 초기 데이터 가져오기 함수 보정
    async function fetchInitialData() {
      const { data, error } = await supabase
        .from('station') // 단수형 고친 테이블 이름
        .select('*')
        .order('id', { ascending: true });

      if (error) {
        console.error('Supabase에서 데이터를 가져오는 중 실패:', error);
        return;
      }

      if (data && data.length > 0) {
        console.log('Supabase 데이터 로드 성공:', data);
        
        // 🌟 Svelte 5 핵심: 완벽하게 가공된 새 배열을 '통째로 할당'해야 반응형이 작동해!
        umbrellas = data.map(item => ({
          id: item.id,
          location: item.location,
          current_count: item.current_count,
          // 물리 모빌용 데이터 세팅
          angle: 0,
          angularVelocity: 0
        }));
      } else {
        console.log('Supabase 연결은 성공했으나 테이블이 비어있습니다. 행(Row)을 다시 확인해 보세요.');
      }
    }

    fetchInitialData();

    // 실시간 구독 채널 (단수형 변경 적용)
    const channel = supabase
      .channel('schema-db-changes')
      .on(
        'postgres_changes',
        { event: 'UPDATE', schema: 'public', table: 'station' },
        (payload) => {
          console.log('실시간 변경 감지:', payload.new);
          const targetIndex = umbrellas.findIndex(u => u.id === payload.new.id);
          if (targetIndex !== -1) {
            umbrellas[targetIndex].current_count = payload.new.current_count;
            umbrellas[targetIndex].angularVelocity += 0.2; 
          }
        }
      )
      .subscribe();

    // 물리 루프 실행
    let animationFrame;
    function loop() {
      umbrellas.forEach(u => {
        const gravity = -0.015 * Math.sin(u.angle); 
        u.angularVelocity += gravity;
        u.angularVelocity *= 0.85; 
        u.angle += u.angularVelocity;
      });
      animationFrame = requestAnimationFrame(loop);
    }
    loop();

    return () => {
      supabase.removeChannel(channel);
      cancelAnimationFrame(animationFrame);
    };
  });
  
	let { stations = [] } = $props();
  
	// 배경 이미지 주소 (도트/비 그래픽 이미지 경로로 대체 가능)
	let backgroundImage = $state("/images/raindrop.jpg");
  
	function init() {
	  const activeData = stations;
	  umbrellas = activeData.map((station) => ({
		...station,
		angle: 0,
		angularVelocity: 0
	  }));
	}
  
	function handleScroll(e) {
	  const currentScrollX = e.currentTarget.scrollLeft;
	  const scrollSpeed = currentScrollX - lastScrollX;
	  lastScrollX = currentScrollX;
  
	  umbrellas.forEach(u => {
		u.angularVelocity -= scrollSpeed * 0.00008; 
	  });
	}
  
	onMount(() => {
	  init();
  
	  let animationFrame;
	  function loop() {
		umbrellas.forEach(u => {
		  const gravity = -0.015 * Math.sin(u.angle); 
		  u.angularVelocity += gravity;
		  u.angularVelocity *= 0.85; 
		  u.angle += u.angularVelocity;
		});
  
		animationFrame = requestAnimationFrame(loop);
	  }
  
	  loop();
	  return () => cancelAnimationFrame(animationFrame);
	});
  

    function handleCardClick(index, location) {
        umbrellas[index].angularVelocity += 0.1; 

        console.log(`${location} 상세 페이지로 라우팅 트리거`);
        
        setTimeout(() => {
          window.location.href = `/station/${umbrellas[index].id}`;
        }, 400); // 0.4초 동안 흔들리는 걸 보여주고 이동
    }
  </script>
  
  <main class="glass-dashboard" style="--bg-image: url('{backgroundImage}')">
	<div class="marquee-banner top">
	  <div class="marquee-track">
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
		<span>2026 나-학교-우리 프로젝트, 공공 우산 위치 확인 서비스 ☂️</span>
	  </div>
	</div>
  
	<header class="glass-header">
	  <div class="header-title-box">
		<div class="glass-title">공공 우산<br>위치 확인</div>
		<div class="status-indicator">
		  <div class="heart-beat">☂️</div>
		  <span>실시간 연결됨</span>
		</div>
	  </div>
	  <p class="subtitle">보관소를 터치하면 상세 위치와 안내를 볼 수 있습니다.</p>
	</header>
  
	<div class="dashboard-wrapper" onscroll={handleScroll}>
	  <div class="scroll-content">
		{#each umbrellas as u, i}
		  <button 
			class="umbrella-node" 
			style="transform: rotate({u.angle}rad); --status-color: {u.current_count > 0 ? '#111' : '#5e5e5e'}"
			onclick={() => handleCardClick(i, u.location)}
		  >
			<div class="glass-string"></div>
			
			<div class="glass-card">
			  <div class="glass-frost-texture"></div>
			  
			  <div class="card-top-bar">
				<span class="node-id">실시간 현황</span>
				<span class="status-tag" style="background: rgba(255,255,255,0.2); color: var(--status-color)">
				  {u.current_count > 0 ? '대여 가능' : '수량 없음'}
				</span>
			  </div>
			  
			  <div class="card-body">
				<h2 class="node-title">{u.location}</h2>
				<div class="cozy-count-box">
				  <span class="count-num">{u.current_count}</span>
				  <span class="unit">개 남음</span>
				</div>
			  </div>
			  
			  <a href={u.id}>
				<div class="card-footer-bar">
					<span>자세히 보기</span>
				  </div>
			  </a>

			</div>
		  </button>
		{/each}
	  </div>
	</div>
  
	<div class="marquee-banner bottom">
	  <div class="marquee-track reverse">
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
		<span>🙏 다음 사람을 위해 사용한 우산은 꼭 정거장에 반납해 주세요</span>
	  </div>
	</div>
  </main>
  
  <style>
	/* 1. 요청한 Pretendard-Medium 오프라인 폰트 등록 */
	@font-face {
	  font-family: 'Pretendard';
	  src: url('/fonts/PRETENDARD-MEDIUM.OTF') format('opentype');
	  font-weight: 500;
	  font-style: normal;
	  font-display: swap;
	}
  
	:global(body) {
	  margin: 0; overflow: hidden; background-color: #111;
	}

	*{
		text-decoration: none;
	}
  
	/* 2. 전체 레이아웃 폰트 적용 */
	.glass-dashboard {
	  width: 100vw; height: 100vh;
	  display: flex; flex-direction: column; justify-content: space-between;
	  
	  /* 폰트 교체 완료 */
	  font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, sans-serif;
	  letter-spacing: -0.02em; /* 프리텐다드 폰트는 자간을 살짝 좁혀야 이쁨 */
	  
	  color: #fff; position: relative; box-sizing: border-box;
	  background-image: var(--bg-image);
	  background-size: cover;
	  background-position: center;
	  background-repeat: no-repeat;
      margin: 0;
	}
  
	.glass-dashboard::before {
	  content: ""; position: absolute; inset: 0;
	  background: rgba(0, 0, 0, 0.3); /* 배경 그래픽과 카드 글씨 분리용 딤 */
	  z-index: -1; pointer-events: none;
	}
  
	/* 흐르는 전광판 스타일 */
	.marquee-banner {
	  width: 100%; background: rgba(255, 255, 255, 0.15); color: #fff;
	  backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px);
	  font-size: 0.9rem; font-weight: 500; padding: 12px 0;
	  overflow: hidden; white-space: nowrap;
	  border-top: 1px solid rgba(255, 255, 255, 0.25);
	  border-bottom: 1px solid rgba(255, 255, 255, 0.25);
	  z-index: 5;
	}
	.marquee-track { display: inline-block; animation: marquee 30s linear infinite; }
	.marquee-track.reverse { animation: marquee-rev 30s linear infinite; }
	@keyframes marquee { 0% { transform: translateX(0%); } 100% { transform: translateX(-50%); } }
	@keyframes marquee-rev { 0% { transform: translateX(-50%); } 100% { transform: translateX(0%); } }
  
	/* 감성 헤더 디자인 조율 */
	.glass-header { padding: 40px 30px 0 30px; position: relative; z-index: 2; }
	.header-title-box {
	  display: flex; justify-content: space-between; align-items: center;
	  border-bottom: 1px solid rgba(255, 255, 255, 0.2); padding-bottom: 14px;
	}
	.glass-title { font-size: 1.8rem; font-weight: medium; color: #fff; text-shadow: 0 2px 10px rgba(0,0,0,0.3); }
	.status-indicator {
	  display: flex; align-items: center; gap: 6px;
	  border: 1px solid rgba(255, 255, 255, 0.4); padding: 5px 12px; font-size: 0.8rem;
	  background: rgba(255, 255, 255, 0.15); font-weight: medium; border-radius: 20px;
	  backdrop-filter: blur(4px); color: #fff;
	}
	.heart-beat { animation: beat 0.6s infinite alternate; }
	@keyframes beat { 0% { transform: scale(0.9); } 100% { transform: scale(1.1); } }
	.subtitle { color: rgba(255, 255, 255, 0.7); font-size: 0.85rem; margin-top: 12px; }
  
	/* 가로 스크롤 코어 */
	.dashboard-wrapper {
	  width: 100%; flex-grow: 1; position: relative; z-index: 2;
	  overflow-x: scroll; overflow-y: hidden;
	  display: flex; align-items: center;
	  -webkit-overflow-scrolling: touch;
	}
	.dashboard-wrapper::-webkit-scrollbar { display: none; }
	.scroll-content { display: flex; gap: 40px; padding: 0 60px; min-width: max-content; }
  
	.umbrella-node {
	  background: none; border: none; padding: 0; cursor: pointer;
	  display: flex; flex-direction: column; align-items: center;
	  transform-origin: top center; will-change: transform;
	}
	.glass-string {
	  width: 2px; height: 60px; 
	  background: linear-gradient(to bottom, rgba(255, 255, 255, 0.1), rgba(255,255,255,0.7));
	}
  
	.glass-card {
    width: 230px; 
    background: rgba(255, 255, 255, 0.35); /* 검은 글씨가 잘 보이도록 카드의 투명도를 약간 높임 */
    border: 1px solid rgba(255, 255, 255, 0.4); 
    border-radius: 20px;
    backdrop-filter: blur(25px) saturate(110%);
    -webkit-backdrop-filter: blur(25px) saturate(110%);
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.15);
    position: relative; overflow: hidden;
    display: flex; flex-direction: column; text-align: left;
    
    /* 💥 카드 내부의 기본 폰트 색상을 진한 검은색(#111111)으로 통일 */
    color: #111111; 
  }
  
  .glass-frost-texture {
    position: absolute; inset: 0;
    background-image: radial-gradient(rgba(0, 0, 0, 0.03) 1px, transparent 0); /* 텍스처 입자도 어두운 톤으로 변경 */
    background-size: 4px 4px; opacity: 0.5; pointer-events: none;
  }

  /* 상단 바 텍스트 색상 변경 */
  .card-top-bar {
    padding: 12px 16px; font-size: 0.75rem; font-weight: medium; 
    color: #333333; /* 연한 검은색 */
    display: flex; justify-content: space-between; align-items: center;
    border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  }
  /* 상태 태그의 배경을 조금 더 밝게 해서 검은 글씨가 돋보이게 처리 */
  .status-tag { padding: 3px 10px; border-radius: 20px; font-weight: medium; font-size: 0.7rem; }
  
  .card-body { padding: 25px 20px; flex-grow: 1; z-index: 1; }
  
  /* 위치 타이틀 색상 변경 */
  .node-title {
    font-size: 1.2rem; 
    color: #111111; /* 진한 검은색 */
    margin: 0 0 20px 0;
    font-weight: medium; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }
  
  .cozy-count-box {
    display: flex; align-items: baseline; justify-content: center;
    margin: 10px 0;
  }
  
  /* 숫자는 고유의 신호등 컬러(--status-color)를 유지하되 가독성을 위해 그림자 제거 */
  .count-num {
    font-size: 4.5rem; font-weight: normal;
    color: var(--status-color);
    line-height: 1;
  }
  /* '개 남음' 단위 텍스트 색상 변경 */
  .unit { 
    font-size: 1.1rem; 
    color: #222222; /* 검은색 계열 */
    margin-left: 6px; 
    font-weight: medium; 
  }

  /* 하단 버튼 영역 색상 변경 */
  .card-footer-bar {
    padding: 14px; border-top: 1px solid rgba(0, 0, 0, 0.1);
    background: rgba(0, 0, 0, 0.03);
    font-size: 0.85rem; 
    color: #222222; /* 검은색 계열 */
    text-align: center;
    font-weight: bold; z-index: 1; transition: all 0.2s;
  }
  /* 마우스 올렸을 때도 선명한 검은색 유지 */
  .glass-card:hover .card-footer-bar { 
    color: #000000; 
    background: rgba(0, 0, 0, 0.08); 
  }
</style>