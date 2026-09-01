<script setup>
import { ref, onMounted, onBeforeUnmount, watch, computed } from 'vue';

const props = defineProps({
  events: {
    type: Array,
    required: true
  },
  selectedEvent: {
    type: Object,
    default: null
  },
  initialTab: {
    type: String,
    default: 'checkin'
  },
  initialEvent: {
    type: Object,
    default: null
  }
});

const emit = defineEmits(['update-event', 'close-scanner']);

const currentEvent = ref(props.selectedEvent || (props.events.length > 0 ? props.events[0] : null));

watch(() => props.selectedEvent, (newEv) => {
  if (newEv) {
    currentEvent.value = newEv;
  }
});

// Setup Lottie Player script if needed
onMounted(() => {
  if (!window.LottiePlayer && !document.getElementById('lottie-player-script')) {
    const script = document.createElement('script');
    script.id = 'lottie-player-script';
    script.src = 'https://unpkg.com/@lottiefiles/lottie-player@latest/dist/lottie-player.js';
    document.head.appendChild(script);
  }
});

const isScanning = ref(false);
const activeTab = ref(props.initialTab === 'report' ? 'report' : 'checkin');

watch(() => props.initialTab, (newTab) => {
  if (newTab) {
    activeTab.value = newTab === 'report' ? 'report' : 'checkin';
  }
});

const openScanner = (event) => {
  currentEvent.value = event;
  isScanning.value = true;
  startCamera();
};

const closeScannerView = () => {
  isScanning.value = false;
  stopCamera();
};

// Scan Results Popup Notification state
const scanPopupVisible = ref(false);
const isConfirmed = ref(false);
const scanResultStatus = ref('success'); // 'success', 'failed', 'already'
const scanData = ref({
  invoice: 'INV-88419',
  name: 'Ahmad Fauzi',
  ticketType: 'E-Ticket',
  time: ''
});

let popupTimer = null;

const triggerScan = (status, rawData = null) => {
  scanResultStatus.value = status;
  isConfirmed.value = false;
  scanPopupVisible.value = true;
  
  const tType = ticketCategory.value === 'invitation' ? 'Invitation' : 'E-Ticket';
  const tCode = rawData ? rawData.substring(0, 10).toUpperCase() : `INV-${Math.floor(10000 + Math.random() * 90000)}`;

  const names = ['Ahmad Fauzi', 'Budi Santoso', 'Citra Kirana', 'Dewi Lestari', 'Eko Prasetyo', 'Fina Melati'];
  let sum = 0;
  for(let i=0; i<tCode.length; i++) sum += tCode.charCodeAt(i);
  const buyerName = names[sum % names.length];

  scanData.value = {
    invoice: tCode,
    name: status === 'failed' ? 'Tidak Terdaftar' : buyerName,
    ticketType: tType,
    time: new Date().toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' }) + ' WIB'
  };
  
  if (popupTimer) clearTimeout(popupTimer);
  popupTimer = setTimeout(() => {
    scanPopupVisible.value = false;
  }, 15000);
};

const closePopup = () => {
  scanPopupVisible.value = false;
  if (popupTimer) clearTimeout(popupTimer);
  if (!isManualMode()) {
    if (animationFrameId) clearTimeout(animationFrameId);
    animationFrameId = setTimeout(scanQRCode, 100);
  }
};

const confirmCheckin = () => {
  if (scanResultStatus.value === 'success' && !isConfirmed.value) {
    isConfirmed.value = true;
    const activeEv = props.selectedEvent || currentEvent.value;
    if (activeEv && activeEv.sold < activeEv.total) {
      activeEv.sold++;
    }
    if (popupTimer) clearTimeout(popupTimer);
    popupTimer = setTimeout(() => {
      closePopup();
    }, 15000);
  } else {
    closePopup();
  }
};

// Camera Scanner View State
const videoElement = ref(null);
let videoStream = null;
const checkinMode = ref('qr'); // 'qr' | 'manual'
const ticketCategory = ref('e-ticket'); // 'e-ticket' | 'invitation'
const manualTicketCode = ref('');

// Derive ticket category from mode
const isManualMode = () => checkinMode.value === 'manual';
const isQRMode = () => checkinMode.value !== 'manual';

// jsQR dynamic script loader
let jsScriptLoaded = false;
const loadJsQR = () => {
  return new Promise((resolve, reject) => {
    if (window.jsQR || jsScriptLoaded) {
      resolve();
      return;
    }
    const script = document.createElement('script');
    script.src = 'https://cdn.jsdelivr.net/npm/jsqr@1.4.0/dist/jsQR.min.js';
    script.onload = () => {
      jsScriptLoaded = true;
      resolve();
    };
    script.onerror = reject;
    document.head.appendChild(script);
  });
};

const cameraErrorPopupVisible = ref(false);
let canvasElement = null;
let canvasCtx = null;
let animationFrameId = null;

const scanQRCode = () => {
  if (videoElement.value && videoElement.value.readyState === videoElement.value.HAVE_ENOUGH_DATA) {
    if (!canvasElement) {
      canvasElement = document.createElement('canvas');
      canvasCtx = canvasElement.getContext('2d', { willReadFrequently: true });
    }
    canvasElement.width = videoElement.value.videoWidth;
    canvasElement.height = videoElement.value.videoHeight;
    canvasCtx.drawImage(videoElement.value, 0, 0, canvasElement.width, canvasElement.height);
    const imageData = canvasCtx.getImageData(0, 0, canvasElement.width, canvasElement.height);
    const code = window.jsQR ? window.jsQR(imageData.data, imageData.width, imageData.height, {
      inversionAttempts: 'attemptBoth'
    }) : null;
    
    if (code) {
      const qrData = code.data;
      
      const statuses = ['success', 'failed', 'already'];
      let hash = 0;
      for (let i = 0; i < qrData.length; i++) {
        hash = qrData.charCodeAt(i) + ((hash << 5) - hash);
      }
      const statusIndex = Math.abs(hash) % statuses.length;
      const status = statuses[statusIndex];
      
      triggerScan(status, qrData);
      
      if (navigator.vibrate) {
        navigator.vibrate(200);
      }
      return;
    }
  }
  
  if (checkinMode.value === 'qr' && !scanPopupVisible.value) {
    animationFrameId = setTimeout(scanQRCode, 100);
  }
};

const startCamera = async () => {
  try {
    cameraErrorPopupVisible.value = false;
    await loadJsQR();
    const stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment' }
    });
    if (videoElement.value) {
      videoElement.value.srcObject = stream;
      videoStream = stream;
      if (animationFrameId) clearTimeout(animationFrameId);
      animationFrameId = setTimeout(scanQRCode, 100);
    }
  } catch (err) {
    console.error("Camera access blocked or error:", err);
    cameraErrorPopupVisible.value = true;
  }
};

const retryCamera = () => {
  cameraErrorPopupVisible.value = false;
  startCamera();
};

const stopCamera = () => {
  cameraErrorPopupVisible.value = false;
  if (animationFrameId) {
    clearTimeout(animationFrameId);
    animationFrameId = null;
  }
  if (videoStream) {
    videoStream.getTracks().forEach(track => track.stop());
    videoStream = null;
  }
};

// Flashlight toggle control
const isFlashlightOn = ref(false);
watch(isFlashlightOn, async (on) => {
  if (videoStream && isQRMode()) {
    const track = videoStream.getVideoTracks()[0];
    if (track) {
      try {
        await track.applyConstraints({
          advanced: [{ torch: on }]
        });
      } catch (err) {
        console.warn("Torch not supported on this device/browser:", err);
      }
    }
  }
});

const handleViewfinderTap = () => {
  const statuses = ['success', 'failed', 'already'];
  const randomStatus = statuses[Math.floor(Math.random() * statuses.length)];
  triggerScan(randomStatus);
  if (navigator.vibrate) {
    navigator.vibrate(200);
  }
};

const handleManualCheckin = () => {
  const code = manualTicketCode.value.trim();
  if (!code) return;
  
  const statuses = ['success', 'failed', 'already'];
  let hash = 0;
  for (let i = 0; i < code.length; i++) {
    hash = code.charCodeAt(i) + ((hash << 5) - hash);
  }
  const statusIndex = Math.abs(hash) % statuses.length;
  const status = statuses[statusIndex];
  
  triggerScan(status, code);
  manualTicketCode.value = '';
  if (navigator.vibrate) {
    navigator.vibrate(200);
  }
};

onBeforeUnmount(() => {
  stopCamera();
});

// ==========================================
// CHECK-IN REPORT TAB STATE & LOGIC
// ==========================================
const selectedReportEvent = ref(props.selectedEvent || (props.events.length > 0 ? props.events[0] : null));
const showEventPicker = ref(false);
const searchQuery = ref('');
const ticketType = ref('eticket'); // 'eticket' | 'invitation'
const isLoading = ref(false);

const stats = computed(() => {
  const ev = selectedReportEvent.value;
  const paid = ev ? (ev.total || 1166) : 1166;
  const checkedIn = ev ? (ev.sold || 949) : 949;
  return {
    paid,
    checkedIn
  };
});

// Sample ticket dataset
const sampleTickets = ref([
  { id: 1, invoice: 'KL-1788282174BXHQPO', name: 'widya nur', ticketNumber: 'ETKT-99214', type: 'VIP Reguler', phone: '081234567890', email: 'widya@mail.com', isCheckin: true, time: '14:20 WIB', ticketKind: 'eticket' },
  { id: 2, invoice: 'KL-1788280767RYCW15', name: 'Jozevin I', ticketNumber: 'ETKT-99215', type: 'General', phone: '081987654321', email: 'jozevin@mail.com', isCheckin: true, time: '14:25 WIB', ticketKind: 'eticket' },
  { id: 3, invoice: 'KL-17882794831TRNKH', name: 'MUHAMMAD', ticketNumber: 'ETKT-99216', type: 'General', phone: '085712345678', email: 'muhammad@mail.com', isCheckin: false, time: null, ticketKind: 'eticket' },
  { id: 4, invoice: 'KL-17882791234ABCD', name: 'Siti Rahma', ticketNumber: 'ETKT-99217', type: 'VIP Reguler', phone: '081399887766', email: 'siti@mail.com', isCheckin: true, time: '15:02 WIB', ticketKind: 'eticket' },
  { id: 5, invoice: 'KL-17882795678EFGH', name: 'Andi Wijaya', ticketNumber: 'INV-10021', type: 'VVIP Guest', phone: '082155443322', email: 'andi@mail.com', isCheckin: false, time: null, ticketKind: 'invitation' },
  { id: 6, invoice: 'KL-17882799999IJKL', name: 'Rina Maryana', ticketNumber: 'INV-10022', type: 'VVIP Guest', phone: '087811223344', email: 'rina@mail.com', isCheckin: true, time: '13:45 WIB', ticketKind: 'invitation' }
]);

const fetchTicketsFromAPI = async () => {
  if (!selectedReportEvent.value) return;
  isLoading.value = true;
  currentPage.value = 1;
  const token = localStorage.getItem('auth_token');
  const apiUrl = import.meta.env.VITE_URL_API || 'https://api.kolektix.my.id';
  try {
    if (ticketType.value === 'invitation') {
      const response = await fetch(`${apiUrl}/api/invitations/event/${selectedReportEvent.value.id}?with_details=true`, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        }
      });
      const result = await response.json();
      const rawData = result.data || result.invitations || result;
      const raw = Array.isArray(rawData) ? rawData : (rawData ? [rawData] : []);
      if (response.ok && raw.length > 0) {
        const parsed = [];
        raw.forEach(inv => {
          const details = inv.event_invitation_detail || inv.has_detail_invitation || inv.event_invitation_details || inv.details || [];
          if (details.length === 0) {
            parsed.push({
              id: inv.id,
              name: inv.invitation_title || 'Invitation',
              invoice: inv.slug || `INV-${inv.id}`,
              ticketNumber: inv.slug || `INV-${inv.id}`,
              seat: '-',
              type: inv.invitation_title || 'Invitation',
              phone: '-',
              email: '-',
              isCheckin: !!inv.is_checkin,
              time: inv.checkin_date ? new Date(inv.checkin_date).toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' }) : '',
              ticketKind: 'invitation'
            });
            return;
          }
          details.forEach(d => {
            parsed.push({
              id: d.id,
              name: d.fullname || d.full_name || inv.invitation_title || 'Guest',
              invoice: inv.slug || `INV-${inv.id}`,
              ticketNumber: d.invitation_number,
              seat: d.seat_number || '-',
              type: inv.invitation_title || 'Invitation',
              phone: d.phone || '-',
              email: d.email || '-',
              isCheckin: d.is_checkin === 1,
              time: d.checkin_date ? new Date(d.checkin_date).toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' }) : '',
              ticketKind: 'invitation'
            });
          });
        });
        if (parsed.length > 0) sampleTickets.value = parsed;
      }
    } else {
      const response = await fetch(`${apiUrl}/api/list-transaction-by-event?event_id=${selectedReportEvent.value.id}&page=1&per_page=1000`, {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Accept': 'application/json'
        }
      });
      const result = await response.json();
      if (response.ok && result.data && result.data.length > 0) {
        const parsed = [];
        result.data.forEach(tr => {
          const etickets = tr.etickets || [];
          const identities = tr.identities || [];
          etickets.forEach(et => {
            const pemesan = identities.find(id => id.is_pemesan === 1) || identities[0] || {};
            parsed.push({
              id: et.id,
              name: pemesan.full_name || tr.has_user?.name || 'Guest',
              invoice: tr.invoice_no,
              ticketNumber: et.eticket_number,
              seat: et.seat_number || '-',
              type: et.has_event_ticket?.name || 'Tiket',
              phone: pemesan.no_telp || tr.has_user?.phone || '-',
              email: pemesan.email || tr.has_user?.email || '-',
              isCheckin: et.is_checkin === 1,
              time: et.checkin_date ? new Date(et.checkin_date).toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' }) : '',
              ticketKind: 'eticket'
            });
          });
        });
        if (parsed.length > 0) sampleTickets.value = parsed;
      }
    }
  } catch (err) {
    console.warn('API fetch warning, using fallback tickets dataset:', err);
  } finally {
    isLoading.value = false;
  }
};

watch([selectedReportEvent, ticketType], () => {
  fetchTicketsFromAPI();
}, { immediate: true });

const filteredTickets = computed(() => {
  return sampleTickets.value.filter(t => {
    const matchesType = (ticketType.value === 'eticket' && t.ticketKind === 'eticket') || (ticketType.value === 'invitation' && t.ticketKind === 'invitation');
    const q = searchQuery.value.toLowerCase().trim();
    const matchesSearch = !q || t.invoice.toLowerCase().includes(q) || t.name.toLowerCase().includes(q) || t.phone.includes(q);
    return matchesType && matchesSearch;
  });
});

// Pagination
const currentPage = ref(1);
const perPage = ref(5);
const totalPages = computed(() => Math.ceil(filteredTickets.value.length / perPage.value) || 1);

const pageNumbers = computed(() => {
  const pages = [];
  for (let i = 1; i <= totalPages.value; i++) {
    pages.push(i);
  }
  return pages;
});

const paginatedTickets = computed(() => {
  const start = (currentPage.value - 1) * perPage.value;
  return filteredTickets.value.slice(start, start + perPage.value);
});

const goToPage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
  }
};

const selectReportEvent = (ev) => {
  selectedReportEvent.value = ev;
  showEventPicker.value = false;
};

const downloadInvoice = (inv) => {
  alert(`Download Invoice: ${inv}`);
};

// Confirm Check-In Modal State
const confirmCheckInVisible = ref(false);
const ticketToCheckIn = ref(null);
const isProcessingCheckIn = ref(false);

const requestCheckIn = (ticket) => {
  ticketToCheckIn.value = ticket;
  confirmCheckInVisible.value = true;
};

const cancelCheckIn = () => {
  if (!isProcessingCheckIn.value) {
    confirmCheckInVisible.value = false;
    ticketToCheckIn.value = null;
  }
};

const confirmReportCheckIn = () => {
  if (!ticketToCheckIn.value) return;
  isProcessingCheckIn.value = true;
  setTimeout(() => {
    ticketToCheckIn.value.isCheckin = true;
    ticketToCheckIn.value.time = new Date().toLocaleTimeString('id-ID', { hour: '2-digit', minute: '2-digit' }) + ' WIB';
    isProcessingCheckIn.value = false;
    confirmCheckInVisible.value = false;
    ticketToCheckIn.value = null;
  }, 600);
};

// Modal Touch Dragging
const pickerStyle = ref({});
let startY = 0;
let currentY = 0;
const handleDragStart = (e) => {
  startY = e.touches ? e.touches[0].clientY : e.clientY;
};
const handleDragMove = (e) => {
  currentY = e.touches ? e.touches[0].clientY : e.clientY;
  const delta = currentY - startY;
  if (delta > 0) {
    pickerStyle.value = { transform: `translateY(${delta}px)` };
  }
};
const handleDragEnd = () => {
  const delta = currentY - startY;
  if (delta > 100) {
    showEventPicker.value = false;
  }
  pickerStyle.value = {};
};
</script>

<template>
  <div class="checkin-container">
    <!-- EVENT LIST PAGE -->
    <div v-if="!isScanning" class="checkin-list-page">
      <!-- Tab Header Buttons -->
      <div class="checkin-tabs-header">
        <button
          class="checkin-tab-btn"
          :class="{ active: activeTab === 'checkin' }"
          @click="activeTab = 'checkin'"
        >
          Check-In
        </button>
        <button
          class="checkin-tab-btn"
          :class="{ active: activeTab === 'report' }"
          @click="activeTab = 'report'"
        >
          Check-In Report
        </button>
      </div>

      <div v-if="activeTab === 'checkin'" class="checkin-events-list">
        <div v-for="event in events" :key="event.id" class="event-card">
          <!-- Thumbnail wrapper -->
          <div class="card-thumbnail-wrapper">
            <img :src="event.image" :alt="event.title" class="event-thumbnail" />
            <div class="status-badge" :class="event.status.toLowerCase()">
              <span class="status-dot"></span>
              <span>{{ event.status === 'Draft' ? 'Draf' : event.status }}</span>
            </div>
          </div>

          <!-- Card Info -->
          <div class="card-info">
            <div class="title-marquee-wrapper">
              <h3 class="event-card-title" :class="{ 'marquee-animate': event.title && event.title.length > 22 }">
                <span>{{ event.title }}</span>
                <span v-if="event.title && event.title.length > 22">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{{ event.title }}</span>
              </h3>
            </div>

            <div class="creator-profile-row" v-if="event.organizer">
              <img :src="event.creatorLogo || 'https://images.unsplash.com/photo-1534528741775-53994a69daeb?w=100&auto=format&fit=crop&q=80'" alt="Creator Profile" class="creator-avatar" />
              <span class="creator-name">{{ event.organizer }}</span>
              <span class="verified-badge">
                <svg viewBox="0 0 24 24" fill="currentColor" class="verified-check-svg">
                  <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
                </svg>
              </span>
            </div>

            <div class="meta-row" v-if="event.location">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="meta-icon">
                <path fill-rule="evenodd" d="m9.69 18.933.003.001C9.89 19.02 10 19 10 19s.11.02.308-.066l.002-.001.006-.003.018-.008a5.741 5.741 0 0 0 .281-.14c.186-.096.446-.24.757-.433.62-.384 1.445-.966 2.274-1.765C15.302 14.988 17 12.493 17 9A7 7 0 1 0 3 9c0 3.492 1.698 5.988 3.343 7.587.829.799 1.655 1.381 2.274 1.765.31.193.57.337.757.433.107.054.2.096.28.14a.515.515 0 0 0 .036.017l.006.003ZM10 12a3 3 0 1 0 0-6 3 3 0 0 0 0 6Z" clip-rule="evenodd" />
              </svg>
              <span class="meta-text">{{ event.location }}</span>
            </div>

            <div class="meta-row" v-if="event.date">
              <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="meta-icon">
                <path fill-rule="evenodd" d="M5.75 2a.75.75 0 0 1 .75.75V4h7V2.75a.75.75 0 0 1 1.5 0V4h.25A2.75 2.75 0 0 1 18 6.75v8.5A2.75 2.75 0 0 1 15.25 18H4.75A2.75 2.75 0 0 1 2 15.25v-8.5A2.75 2.75 0 0 1 4.75 4H5V2.75A.75.75 0 0 1 5.75 2Zm-1 5.5c-.69 0-1.25.56-1.25 1.25v6.5c0 .69.56 1.25 1.25 1.25h10.5c.69 0 1.25-.56 1.25-1.25v-6.5c0-.69-.56-1.25-1.25-1.25H4.75Z" clip-rule="evenodd" />
              </svg>
              <span class="meta-text">{{ event.date }}</span>
            </div>

            <div class="price-row" v-if="event.price">
              <span class="event-card-price">{{ event.price }}</span>
            </div>

            <div class="card-footer-row" v-if="event.total">
              <div class="ticket-sales-info">
                <div class="sales-text-row">
                  <span class="sales-text">{{ event.sold }}/{{ event.total }} Tiket Terjual</span>
                  <span class="sales-percent">{{ Math.round((event.sold / event.total) * 100) }}%</span>
                </div>
                <div class="sales-progress-bar">
                  <div
                    class="sales-progress-fill"
                    :style="{ width: `${(event.sold / event.total) * 100}%` }"
                  ></div>
                </div>
              </div>
            </div>

            <div class="checkin-action-wrapper" style="margin-top: 10px;">
              <button class="checkin-card-action-btn" @click="openScanner(event)">
                Check-In
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Report Tab View -->
      <div v-else class="checkin-report-view">
        <div class="report-container">
          <!-- Top Three-Column Stats Box -->
          <div class="stats-overview-card" v-if="selectedReportEvent">
            <div class="overview-col">
              <span class="overview-label">Tiket Terjual</span>
              <span class="overview-val">{{ stats.paid }}</span>
            </div>
            <div class="overview-divider"></div>
            <div class="overview-col">
              <span class="overview-label">Sudah Check-In</span>
              <span class="overview-val">{{ stats.checkedIn }}</span>
            </div>
            <div class="overview-divider"></div>
            <div class="overview-col">
              <span class="overview-label">Belum Check-In</span>
              <span class="overview-val">{{ stats.paid - stats.checkedIn }}</span>
            </div>
          </div>

          <!-- Top Event Selector Button -->
          <button class="select-event-btn" @click="showEventPicker = true">
            <div class="btn-left">
              <svg class="calendar-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="3" y="4" width="18" height="16" rx="2" stroke-linecap="round" stroke-linejoin="round" />
                <line x1="3" y1="9" x2="21" y2="9" stroke-linecap="round" stroke-linejoin="round" />
              </svg>
              <span class="event-btn-title">{{ selectedReportEvent ? selectedReportEvent.title : 'Pilih Event' }}</span>
            </div>
            <svg class="chevron-right-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
              <polyline points="9 18 15 12 9 6"></polyline>
            </svg>
          </button>

          <!-- Search Field -->
          <div class="search-wrapper">
            <svg class="search-icon" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" d="m21 21-5.197-5.197m0 0A7.5 7.5 0 1 0 5.196 5.196a7.5 7.5 0 0 0 10.607 10.607Z" />
            </svg>
            <input type="text" v-model="searchQuery" placeholder="Cari (Invoice, Nama, HP...)" class="search-input" />
          </div>

          <!-- Segmented Control -->
          <div class="segmented-control">
            <button class="segment-btn" :class="{ active: ticketType === 'eticket' }" @click="ticketType = 'eticket'">E-Ticket</button>
            <button class="segment-btn" :class="{ active: ticketType === 'invitation' }" @click="ticketType = 'invitation'">Invitation</button>
          </div>

          <!-- Stats summary line -->
          <div class="summary-info-row" v-if="selectedReportEvent">
            <span class="total-text">Menampilkan {{ filteredTickets.length }} tiket</span>
            <div class="checked-in-count">
              <span class="green-dot"></span>
              <span>{{ stats.checkedIn }} Checked In</span>
            </div>
          </div>

          <!-- Tickets List Table (Scrollable X) -->
          <div class="table-container-wrapper">
            <div v-if="isLoading" class="loading-state">
              Memuat data tiket...
            </div>
            <div v-else-if="filteredTickets.length === 0" class="empty-tickets">
              <p class="empty-label">Tidak ada tiket ditemukan</p>
            </div>
            <div v-else class="table-scroll-x">
              <table class="report-table">
                <thead>
                  <tr>
                    <th>No</th>
                    <th>No Invoice</th>
                    <th>Nama Pembeli</th>
                    <th>No. E-Ticket</th>
                    <th>Tipe Tiket</th>
                    <th>Telepon</th>
                    <th>Email</th>
                    <th>Status</th>
                    <th>Waktu</th>
                    <th>Aksi</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(ticket, index) in paginatedTickets" :key="ticket.id">
                    <td>{{ (currentPage - 1) * perPage + index + 1 }}</td>
                    <td class="clickable-invoice" @click="downloadInvoice(ticket.invoice)">{{ ticket.invoice }}</td>
                    <td>{{ ticket.name }}</td>
                    <td>{{ ticket.ticketNumber }}</td>
                    <td>{{ ticket.type }}</td>
                    <td>{{ ticket.phone }}</td>
                    <td>{{ ticket.email }}</td>
                    <td>
                      <span class="status-badge-report" :class="{ 'badge-done': ticket.isCheckin, 'badge-wait': !ticket.isCheckin }">
                        {{ ticket.isCheckin ? 'Sudah Check-In' : 'Belum Check-In' }}
                      </span>
                    </td>
                    <td>{{ ticket.time || '-' }}</td>
                    <td>
                      <button v-if="!ticket.isCheckin" class="table-action-btn" @click="requestCheckIn(ticket)">
                        Check In
                      </button>
                      <span v-else class="done-text">Selesai</span>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>

          <!-- Pagination Controls -->
          <div v-if="!isLoading && filteredTickets.length > 0" class="pagination-wrapper">
            <button class="page-btn" :disabled="currentPage === 1" @click="goToPage(1)">«</button>
            <button class="page-btn" :disabled="currentPage === 1" @click="goToPage(currentPage - 1)">‹</button>
            <button
              v-for="p in pageNumbers"
              :key="p"
              class="page-btn"
              :class="{ active: p === currentPage }"
              @click="goToPage(p)"
            >{{ p }}</button>
            <button class="page-btn" :disabled="currentPage === totalPages" @click="goToPage(currentPage + 1)">›</button>
            <button class="page-btn" :disabled="currentPage === totalPages" @click="goToPage(totalPages)">»</button>
            <span class="page-info">Halaman {{ currentPage }} / {{ totalPages }}</span>
          </div>

          <!-- Draggable Event Picker Modal -->
          <transition name="modal-fade">
            <div class="modal-overlay" v-if="showEventPicker" @click.self="showEventPicker = false">
              <div 
                class="modal-sheet"
                :style="pickerStyle"
                @touchstart="handleDragStart($event)"
                @touchmove="handleDragMove"
                @touchend="handleDragEnd"
                @mousedown="handleDragStart($event)"
              >
                <div class="modal-handle"></div>
                <div class="detail-modal-header">
                  <h3 class="modal-title">Pilih Event</h3>
                  <button class="close-x-btn" @click="showEventPicker = false">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="close-icon">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M6 18 18 6M6 6l12 12" />
                    </svg>
                  </button>
                </div>

                <div class="modal-list modal-event-list">
                  <div
                    v-for="ev in events"
                    :key="ev.id"
                    class="modal-item"
                    :class="{ 'selected': selectedReportEvent && selectedReportEvent.id === ev.id }"
                    @click="selectReportEvent(ev)"
                  >
                    <span>{{ ev.title }}</span>
                  </div>
                </div>
              </div>
            </div>
          </transition>

          <!-- Confirm Check-in Modal -->
          <transition name="modal-fade">
            <div class="modal-overlay" v-if="confirmCheckInVisible && ticketToCheckIn" @click.self="cancelCheckIn">
              <div class="modal-sheet confirm-sheet">
                <div class="modal-handle"></div>
                <div class="detail-modal-header">
                  <h3 class="modal-title">Konfirmasi Check-In</h3>
                  <button class="close-x-btn" @click="cancelCheckIn" :disabled="isProcessingCheckIn">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="close-icon">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M6 18 18 6M6 6l12 12" />
                    </svg>
                  </button>
                </div>

                <div class="confirm-body" v-if="ticketToCheckIn">
                  <p class="confirm-text">
                    Apakah Anda yakin ingin melakukan check-in untuk data di bawah ini?
                  </p>
                  <div class="confirm-row">
                    <span class="confirm-label">No. {{ ticketToCheckIn.ticketKind === 'invitation' ? 'Invitation' : 'E-Ticket' }}</span>
                    <span class="confirm-val">{{ ticketToCheckIn.ticketNumber }}</span>
                  </div>
                  <div class="confirm-row">
                    <span class="confirm-label">Nama</span>
                    <span class="confirm-val">{{ ticketToCheckIn.name }}</span>
                  </div>
                  <div class="confirm-row">
                    <span class="confirm-label">Tipe</span>
                    <span class="confirm-val">{{ ticketToCheckIn.type }}</span>
                  </div>
                </div>

                <div class="confirm-actions">
                  <button class="confirm-btn-secondary" @click="cancelCheckIn" :disabled="isProcessingCheckIn">
                    Batal
                  </button>
                  <button class="confirm-btn-primary" @click="confirmReportCheckIn" :disabled="isProcessingCheckIn">
                    {{ isProcessingCheckIn ? 'Memproses...' : 'Ya, Check-In' }}
                  </button>
                </div>
              </div>
            </div>
          </transition>
        </div>
      </div>
    </div>

    <!-- SCANNER PAGE -->
    <div v-else class="scanner-page">
      <div class="scanner-fullscreen-container" :class="{ 'manual-bg': checkinMode === 'manual' }">
        <!-- Real Camera Feed Video Element -->
        <video v-show="isQRMode()" ref="videoElement" autoplay playsinline class="scanner-camera-feed"></video>
        
        <!-- Scanner Top Bar -->
        <div class="scanner-top-bar">
          <button
            class="scanner-icon-btn close-btn"
            :class="{ 'manual-mode': checkinMode === 'manual' }"
            @click="closeScannerView"
          >
            <!-- Close X Icon -->
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" class="scanner-svg-icon">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>

          <div class="scanner-header-title">
            <div class="marquee-wrap">
              <span class="marquee-track" :class="{ 'marquee-animate': ((currentEvent ? currentEvent.title : 'All Event') + ' - ' + (ticketCategory === 'invitation' ? 'Invitation' : 'E-Ticket')).length > 18 }">
                {{ currentEvent ? currentEvent.title : 'All Event' }} - {{ ticketCategory === 'invitation' ? 'Invitation' : 'E-Ticket' }}
                <template v-if="((currentEvent ? currentEvent.title : 'All Event') + ' - ' + (ticketCategory === 'invitation' ? 'Invitation' : 'E-Ticket')).length > 18">
                  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
                  {{ currentEvent ? currentEvent.title : 'All Event' }} - {{ ticketCategory === 'invitation' ? 'Invitation' : 'E-Ticket' }}
                </template>
              </span>
            </div>
            <p>{{ currentEvent ? currentEvent.location : 'Semua Lokasi' }}</p>
          </div>
          
          <!-- Flash button on right -->
          <button v-show="isQRMode()" class="scanner-icon-btn flash-btn" :class="{ active: isFlashlightOn }" @click="isFlashlightOn = !isFlashlightOn">
            <!-- Flash Bolt Icon -->
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" class="scanner-svg-icon" :style="{ fill: isFlashlightOn ? '#F5C453' : 'none', stroke: isFlashlightOn ? '#F5C453' : 'currentColor' }">
              <polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2" />
            </svg>
          </button>
          <!-- Spacer when flash hidden (manual mode) to keep title centered -->
          <div v-show="!isQRMode()" style="width:40px;"></div>
        </div>
        
        <!-- Check-in Mode Selector Tab Toggle -->
        <div class="checkin-mode-toggle">
          <button class="toggle-btn" :class="{ active: checkinMode === 'qr' }" @click="checkinMode = 'qr'">
            Scan-QR
          </button>
          <button class="toggle-btn" :class="{ active: checkinMode === 'manual' }" @click="checkinMode = 'manual'">
            Scanner
          </button>
        </div>
        
        <!-- Scanner Middle Body Content Area -->
        <div class="scanner-body">
          <!-- Scanning Window Area (QR Mode: Reguler & Invitation) -->
          <div v-if="isQRMode()" class="scanner-viewfinder" @click="handleViewfinderTap">
            <div class="scanner-laser-trail"></div>
            <div class="scanner-laser-line"></div>
          </div>
          
          <!-- Manual Input Card (Manual Mode) -->
          <div v-if="checkinMode === 'manual'" class="manual-checkin-container">
            <div class="manual-form-card">
              <div class="manual-field-group">
                <label class="manual-label">Kode Tiket / Invoice</label>
                <input 
                  v-model="manualTicketCode" 
                  type="text" 
                  placeholder="Masukkan kode tiket" 
                  class="manual-input-field"
                  @keyup.enter="handleManualCheckin"
                />
                <p class="manual-desc-mockup">Masukkan nomor invoice atau kode tiket untuk diproses scanner</p>
              </div>
              
              <button class="manual-submit-btn" @click="handleManualCheckin">
                Check-in Tiket
              </button>
              
              <div class="manual-tips">
                <span>💡 <strong>Tips Simulasi:</strong> Ketik kata "gagal" untuk simulasi gagal, atau "sudah" untuk simulasi tiket terpakai.</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Ticket Category Bottom Bar -->
        <div class="ticket-category-bar">
          <div class="category-buttons-row">
            <button class="sheet-category-btn" :class="{ active: ticketCategory === 'e-ticket' }" @click="ticketCategory = 'e-ticket'">E-Ticket</button>
            <button class="sheet-category-btn" :class="{ active: ticketCategory === 'invitation' }" @click="ticketCategory = 'invitation'">Invitation</button>
          </div>
        </div>

        <!-- Bottom Total Audience Progress Bar -->
        <div class="scanner-bottom-stats" v-if="currentEvent">
          <div class="stats-row">
            <span class="stats-label">TOTAL MASUK</span>
            <span class="stats-val">{{ currentEvent.sold }} / {{ currentEvent.total }} Penonton</span>
          </div>
          <div class="stats-progress-container">
            <div class="stats-progress-track">
              <div class="stats-progress-fill" :style="{ width: `${(currentEvent.sold / currentEvent.total) * 100}%` }"></div>
            </div>
          </div>
        </div>

        <!-- Sliding Notification Popups -->
        <transition name="popup-slide">
          <div v-if="scanPopupVisible" class="notification-popup-card">
            <div class="notification-header" :class="scanResultStatus === 'success' ? (isConfirmed ? 'success' : 'pending') : scanResultStatus">
              <div class="header-icon">
                <svg v-if="scanResultStatus === 'success' && !isConfirmed" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="status-svg">
                  <path fill-rule="evenodd" d="M12 2.25c-5.385 0-9.75 4.365-9.75 9.75s4.365 9.75 9.75 9.75 9.75-4.365 9.75-9.75S17.385 2.25 12 2.25zM12.75 6a.75.75 0 00-1.5 0v6c0 .414.336.75.75.75h4.5a.75.75 0 000-1.5h-3.75V6z" clip-rule="evenodd" />
                </svg>
                <svg v-else-if="scanResultStatus === 'success' && isConfirmed" viewBox="0 0 20 20" fill="currentColor" class="status-svg">
                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.857-9.809a.75.75 0 00-1.214-.882l-3.483 4.79-1.88-1.88a.75.75 0 10-1.06 1.061l2.5 2.5a.75.75 0 001.137-.089l4-5.5z" clip-rule="evenodd"/>
                </svg>
                <svg v-else-if="scanResultStatus === 'failed'" viewBox="0 0 20 20" fill="currentColor" class="status-svg">
                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.28 7.22a.75.75 0 00-1.06 1.06L8.94 10l-1.72 1.72a.75.75 0 101.06 1.06L10 11.06l1.72 1.72a.75.75 0 101.06-1.06L11.06 10l1.72-1.72a.75.75 0 00-1.06-1.06L10 8.94 8.28 7.22z" clip-rule="evenodd"/>
                </svg>
                <svg v-else viewBox="0 0 20 20" fill="currentColor" class="status-svg">
                  <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a.75.75 0 000 1.5h.253a.25.25 0 01.244.304l-.459 2.066A1.75 1.75 0 0010.747 15H11a.75.75 0 000-1.5h-.253a.25.25 0 01-.244-.304l.459-2.066A1.75 1.75 0 009.253 9H9z" clip-rule="evenodd" />
                </svg>
              </div>
              <span class="header-text">
                {{ scanResultStatus === 'success' ? (isConfirmed ? 'BERHASIL CHECK-IN' : 'MENUNGGU KONFIRMASI') : (scanResultStatus === 'failed' ? 'GAGAL SCAN' : 'SUDAH CHECK-IN') }}
              </span>
            </div>

            <div class="notification-body">
              <div class="attendee-row">
                <div class="attendee-info">
                  <p class="event-name">{{ currentEvent ? currentEvent.title : 'Event Kolektix' }}</p>
                  <h4 class="invoice-text">{{ scanData.invoice }}</h4>
                  <p class="buyer-name">{{ scanData.name }}</p>
                  <p class="ticket-type-label">{{ scanData.ticketType }}</p>
                </div>
              </div>

              <div class="scan-time-row">
                <div class="scan-time-info">
                  <span class="time-label">WAKTU MASUK</span>
                  <span class="time-val">{{ scanData.time || 'Baru Saja' }}</span>
                </div>
                <button class="continue-btn" @click="confirmCheckin">
                  {{ scanResultStatus === 'success' && !isConfirmed ? 'Konfirmasi' : (scanResultStatus === 'failed' ? 'Coba Lagi' : 'Lanjut Scan') }}
                </button>
              </div>
            </div>
          </div>
        </transition>

        <!-- Slide-up Camera Failure Error Card -->
        <transition name="popup-slide">
          <div v-if="cameraErrorPopupVisible" class="notification-popup-card camera-error-card">
            <div class="notification-header failed">
              <div class="header-icon">
                <svg viewBox="0 0 20 20" fill="currentColor" class="status-svg">
                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.28 7.22a.75.75 0 00-1.06 1.06L8.94 10l-1.72 1.72a.75.75 0 101.06 1.06L10 11.06l1.72 1.72a.75.75 0 101.06-1.06L11.06 10l1.72-1.72a.75.75 0 00-1.06-1.06L10 8.94 8.28 7.22z" clip-rule="evenodd"/>
                </svg>
              </div>
              <span class="header-text">KAMERA GAGAL DIAKTIFKAN</span>
            </div>
            <div class="notification-body">
              <p class="error-card-desc">Gagal mengakses kamera perangkat. Pastikan izin kamera telah diberikan atau coba gunakan input kode manual sebagai gantinya.</p>
              <div class="error-action-row">
                <button class="continue-btn" @click="retryCamera">Coba Lagi</button>
                <button class="continue-btn manual-switch-btn" @click="checkinMode = 'qr'; cameraErrorPopupVisible = false;">Input Manual</button>
              </div>
            </div>
          </div>
        </transition>
      </div>
    </div>
  </div>
</template>

<style scoped>
.checkin-container {
  width: 100%;
  height: 100%;
}

.checkin-list-page {
  padding: 0 0 100px 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
  background-color: #f8fafc;
  min-height: 100vh;
}

.checkin-tabs-header {
  display: flex;
  border-bottom: 1px solid #e2e8f0;
  width: 100%;
  padding: 0 16px;
  background-color: white;
  position: sticky;
  top: 0;
  z-index: 10;
}

.checkin-tab-btn {
  flex: 1;
  background: none;
  border: none;
  padding: 12px 8px;
  font-size: 13px;
  font-weight: 500;
  color: #64748b;
  cursor: pointer;
  position: relative;
  text-align: center;
  transition: color 0.2s;
  font-family: var(--font-sans);
}

.checkin-tab-btn.active {
  color: var(--primary-base);
  font-weight: 600;
}

.checkin-tab-btn.active::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 3px;
  background-color: var(--primary-base);
  border-radius: 3px 3px 0 0;
}

.checkin-events-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 0 16px;
}

.event-card {
  background-color: var(--white);
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 2px 6px rgba(0,0,0,0.02);
}

.card-thumbnail-wrapper {
  position: relative;
  width: 100%;
  height: 140px;
  background-color: #f1f5f9;
}

.event-thumbnail {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.status-badge {
  position: absolute;
  top: 12px;
  left: 12px;
  background-color: var(--white);
  border-radius: 20px;
  padding: 3px 8px;
  font-size: 10px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 4px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
}

.status-badge.live { color: #16a34a; }
.status-badge.live .status-dot { background-color: #16a34a; }
.status-badge.upcoming { color: #ea580c; }
.status-badge.upcoming .status-dot { background-color: #ea580c; }
.status-dot { width: 6px; height: 6px; border-radius: 50%; display: inline-block; }

.card-info {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.title-marquee-wrapper {
  overflow: hidden;
  white-space: nowrap;
  width: 100%;
  padding-bottom: 2px;
}

.event-card-title {
  font-size: 15px;
  font-weight: 700;
  color: #0f172a;
  margin: 0;
  display: inline-block;
}

.event-card-title.marquee-animate {
  animation: card-title-scroll 12s linear infinite;
}

@keyframes card-title-scroll {
  0% { transform: translateX(0); }
  100% { transform: translateX(-50%); }
}

.creator-profile-row {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 2px;
}

.creator-avatar {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  object-fit: cover;
}

.creator-name {
  font-size: 12px;
  color: #475569;
  font-weight: 500;
}

.verified-check-svg {
  width: 14px;
  height: 14px;
  color: #3b82f6;
}

.meta-row {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: #64748b;
}

.meta-icon {
  width: 14px;
  height: 14px;
  color: #194E9E;
}

.price-row {
  margin-top: 2px;
}

.event-card-price {
  font-size: 14px;
  font-weight: 700;
  color: #0f172a;
}

.card-footer-row {
  margin-top: 4px;
  padding-top: 8px;
  border-top: 1px dashed #e2e8f0;
}

.ticket-sales-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.sales-text-row {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #0f172a;
  font-weight: 600;
}

.sales-progress-bar {
  height: 6px;
  background-color: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.sales-progress-fill {
  height: 100%;
  background-color: #194E9E;
  border-radius: 3px;
}

.checkin-card-action-btn {
  width: 100%;
  background-color: #194E9E;
  color: var(--white);
  border: none;
  border-radius: 6px;
  padding: 10px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(25, 78, 158, 0.12);
}

.checkin-report-view {
  padding: 16px;
  background-color: #f8fafc;
  min-height: 100vh;
}

.report-container {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

/* Stats Overview Card */
.stats-overview-card {
  background: white;
  border-radius: 8px;
  padding: 12px 14px;
  display: flex;
  align-items: center;
  justify-content: space-around;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.03);
  border: 1px solid #e2e8f0;
}

.overview-col {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  flex: 1;
}

.overview-label {
  font-size: 10px;
  color: #64748b;
  font-weight: 500;
  text-align: center;
  white-space: nowrap;
}

.overview-val {
  font-size: 17px;
  font-weight: 800;
  color: #0f172a;
  white-space: nowrap;
}

.overview-divider {
  width: 1px;
  height: 32px;
  background-color: #e2e8f0;
}

/* Select Event Button */
.select-event-btn {
  background-color: #194E9E;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 12px 14px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(25, 78, 158, 0.15);
  transition: transform 0.15s ease, background-color 0.15s ease;
}

.select-event-btn:active {
  transform: scale(0.99);
  background-color: #154287;
}

.btn-left {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
  flex: 1;
}

.calendar-icon {
  width: 18px;
  height: 18px;
  stroke: white;
  flex-shrink: 0;
}

.event-btn-title {
  font-size: 14px;
  font-weight: 700;
  letter-spacing: -0.2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  display: block;
}

.chevron-right-icon {
  width: 18px;
  height: 18px;
  stroke: white;
  flex-shrink: 0;
}

/* Search Wrapper */
.search-wrapper {
  position: relative;
  width: 100%;
}

.search-icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  width: 18px;
  height: 18px;
  color: #94a3b8;
}

.search-input {
  width: 100%;
  background-color: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 12px 14px 12px 42px;
  font-size: 13px;
  color: #0f172a;
  outline: none;
  box-sizing: border-box;
  transition: border-color 0.2s;
}

.search-input:focus {
  border-color: #194E9E;
}

/* Segmented Control */
.segmented-control {
  display: flex;
  background-color: #f1f5f9;
  border-radius: 8px;
  padding: 4px;
  gap: 4px;
}

.segment-btn {
  flex: 1;
  background: none;
  border: none;
  padding: 10px;
  font-size: 13px;
  font-weight: 600;
  color: #64748b;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.segment-btn.active {
  background-color: white;
  color: #194E9E;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

/* Summary Info Row */
.summary-info-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 13px;
  color: #475569;
  font-weight: 500;
  padding: 2px 4px;
}

.checked-in-count {
  display: flex;
  align-items: center;
  gap: 6px;
  color: #0f172a;
  font-weight: 600;
}

.green-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #10b981;
}

/* Table Wrapper */
.table-container-wrapper {
  background-color: white;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.02);
}

.table-scroll-x {
  overflow-x: auto;
  scrollbar-width: thin;
}

.report-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
  font-size: 12px;
  white-space: nowrap;
}

.report-table th {
  background-color: #f8fafc;
  color: #475569;
  font-weight: 700;
  padding: 12px 14px;
  border-bottom: 1px solid #e2e8f0;
}

.report-table td {
  padding: 12px 14px;
  border-bottom: 1px solid #f1f5f9;
  color: #334155;
}

.clickable-invoice {
  color: #194E9E;
  font-weight: 700;
  cursor: pointer;
  text-decoration: underline;
}

.status-badge-report {
  display: inline-block;
  padding: 4px 8px;
  border-radius: 6px;
  font-size: 10px;
  font-weight: 600;
}

.status-badge-report.badge-done {
  background-color: #d1fae5;
  color: #047857;
}

.status-badge-report.badge-wait {
  background-color: #fef3c7;
  color: #b45309;
}

.table-action-btn {
  background-color: #194E9E;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
}

.done-text {
  color: #94a3b8;
  font-size: 11px;
}

/* Pagination Wrapper */
.pagination-wrapper {
  display: flex;
  align-items: center;
  gap: 6px;
  justify-content: center;
  flex-wrap: wrap;
  padding-top: 4px;
}

.page-btn {
  background-color: white;
  border: 1px solid #cbd5e1;
  color: #334155;
  border-radius: 6px;
  padding: 6px 10px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.page-btn.active {
  background-color: #194E9E;
  color: white;
  border-color: #194E9E;
}

.page-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.page-info {
  font-size: 11px;
  color: #64748b;
  margin-left: 6px;
}

/* Modals */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.45);
  z-index: 999;
  display: flex;
  align-items: flex-end;
  justify-content: center;
}

.modal-sheet {
  background: white;
  border-top-left-radius: 20px;
  border-top-right-radius: 20px;
  width: 100%;
  max-width: 480px;
  padding: 16px 20px 24px 20px;
  box-shadow: 0 -4px 20px rgba(0,0,0,0.15);
  box-sizing: border-box;
  animation: slideUp 0.25s ease-out;
}

@keyframes slideUp {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}

.modal-handle {
  width: 36px;
  height: 4px;
  background-color: #cbd5e1;
  border-radius: 2px;
  margin: 0 auto 14px auto;
}

.detail-modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.modal-title {
  font-size: 16px;
  font-weight: 700;
  color: #0f172a;
  margin: 0;
}

.close-x-btn {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 4px;
}

.close-icon {
  width: 20px;
  height: 20px;
}

.modal-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 260px;
  overflow-y: auto;
}

.modal-item {
  padding: 12px 16px;
  border-radius: 10px;
  background-color: #f8fafc;
  font-size: 14px;
  font-weight: 600;
  color: #334155;
  cursor: pointer;
  border: 1px solid #e2e8f0;
}

.modal-item.selected {
  background-color: #eff6ff;
  border-color: #194E9E;
  color: #194E9E;
}

/* Confirm Sheet */
.confirm-body {
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.confirm-text {
  font-size: 13px;
  color: #475569;
  margin: 0 0 8px 0;
}

.confirm-row {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  padding: 6px 0;
  border-bottom: 1px dashed #e2e8f0;
}

.confirm-label {
  color: #64748b;
}

.confirm-val {
  font-weight: 700;
  color: #0f172a;
}

.confirm-actions {
  display: flex;
  gap: 10px;
}

.confirm-btn-secondary {
  flex: 1;
  background-color: #f1f5f9;
  color: #475569;
  border: none;
  border-radius: 10px;
  padding: 12px;
  font-weight: 600;
  cursor: pointer;
}

.confirm-btn-primary {
  flex: 1;
  background-color: #194E9E;
  color: white;
  border: none;
  border-radius: 10px;
  padding: 12px;
  font-weight: 600;
  cursor: pointer;
}

/* Scanner view styles */
.scanner-page {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 2000;
  background-color: #000;
  display: flex;
  flex-direction: column;
}

.scanner-fullscreen-container {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background-color: #000;
  overflow: hidden;
  color: white;
  font-family: var(--font-sans);
}

.scanner-body {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;
  z-index: 3;
}

.scanner-camera-feed {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  z-index: 1;
}

.scanner-top-bar {
  position: relative;
  z-index: 5;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  background: transparent;
}

.scanner-header-title {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  pointer-events: none;
  max-width: 180px;
  overflow: hidden;
}

.marquee-wrap {
  overflow: hidden;
  white-space: nowrap;
}
.marquee-track {
  display: inline-block;
  font-size: 14px;
  font-weight: 700;
  color: white;
  text-shadow: 0 1px 3px rgba(0,0,0,0.5);
}
.marquee-animate {
  animation: marquee-scroll 10s linear infinite;
}
@keyframes marquee-scroll {
  0%   { transform: translateX(0); }
  100% { transform: translateX(-50%); }
}

.scanner-header-title h3 {
  font-size: 14px;
  font-weight: 700;
  margin: 0;
  text-shadow: 0 1px 3px rgba(0,0,0,0.5);
}

.scanner-header-title p {
  font-size: 10px;
  color: rgba(255, 255, 255, 0.7);
  margin: 2px 0 0 0;
  text-shadow: 0 1px 2px rgba(0,0,0,0.5);
}

.scanner-icon-btn {
  background-color: rgba(0, 0, 0, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 50%;
  color: white;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  backdrop-filter: blur(4px);
  transition: background-color 0.2s, border-color 0.2s;
}

.close-btn {
  width: 34px;
  height: 34px;
}
.close-btn .scanner-svg-icon {
  width: 16px;
  height: 16px;
}

.flash-btn {
  width: 30px;
  height: 30px;
}
.flash-btn .scanner-svg-icon {
  width: 14px;
  height: 14px;
}

.close-btn.manual-mode {
  background-color: rgba(25, 78, 158, 0.4);
  border-color: #194E9E;
}

.scanner-svg-icon {
  width: 20px;
  height: 20px;
}

.checkin-mode-toggle {
  position: relative;
  z-index: 3;
  display: flex;
  align-self: center;
  background-color: rgba(255, 255, 255, 0.15);
  padding: 3px;
  border-radius: 30px;
  gap: 4px;
  margin-top: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(8px);
}

.toggle-btn {
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.8);
  padding: 6px 14px;
  font-size: 11px;
  font-weight: 600;
  border-radius: 20px;
  cursor: pointer;
}

.toggle-btn.active {
  background-color: #194E9E;
  color: white;
}

.scanner-viewfinder {
  position: absolute;
  top: -140px;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 3;
  width: 100%;
  height: calc(100% + 140px);
  pointer-events: none;
}

.scanner-laser-line {
  position: absolute;
  left: 0;
  right: 0;
  height: 4px;
  background-color: #194E9E;
  border-radius: 50%;
  box-shadow: 0 0 15px 4px rgba(25, 78, 158, 0.9);
  animation: scan-laser 3s infinite alternate cubic-bezier(0.45, 0.05, 0.55, 0.95);
  z-index: 2;
}

.scanner-laser-trail {
  position: absolute;
  left: 0;
  right: 0;
  height: 400px;
  background: linear-gradient(
    to bottom,
    rgba(25, 78, 158, 0) 0%,
    rgba(25, 78, 158, 0.6) 100%
  );
  animation: 
    scan-laser 3s infinite alternate cubic-bezier(0.45, 0.05, 0.55, 0.95),
    scan-flip 6s infinite step-end,
    scan-opacity 6s infinite ease-in-out;
  z-index: 1;
}

@keyframes scan-laser {
  0% { top: 0%; }
  100% { top: 100%; }
}

@keyframes scan-flip {
  0% { transform: translateY(-100%) scaleY(1); }
  50% { transform: translateY(0%) scaleY(-1); }
  100% { transform: translateY(-100%) scaleY(1); }
}

@keyframes scan-opacity {
  0% { opacity: 0; }
  10% { opacity: 1; }
  40% { opacity: 1; }
  50% { opacity: 0; }
  60% { opacity: 1; }
  90% { opacity: 1; }
  100% { opacity: 0; }
}

.scanner-fullscreen-container.manual-bg {
  background-color: #f8fafc;
  color: #1e293b;
}

.scanner-fullscreen-container.manual-bg .scanner-header-title h3,
.scanner-fullscreen-container.manual-bg .marquee-track {
  color: #1e293b;
  text-shadow: none;
}

.scanner-fullscreen-container.manual-bg .scanner-header-title p {
  color: #64748b;
  text-shadow: none;
}

.scanner-fullscreen-container.manual-bg .scanner-top-bar {
  background: none;
}

.scanner-fullscreen-container.manual-bg .checkin-mode-toggle {
  background-color: rgba(0, 0, 0, 0.05);
  border-color: rgba(0, 0, 0, 0.08);
}

.scanner-fullscreen-container.manual-bg .toggle-btn {
  color: #475569;
}

.scanner-fullscreen-container.manual-bg .toggle-btn.active {
  color: white;
}

.manual-checkin-container {
  position: relative;
  z-index: 3;
  width: 100%;
  padding: 0 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  transform: translateY(-30px);
}

.manual-form-card {
  width: 100%;
  max-width: 320px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  background-color: white;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}

.manual-field-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
  text-align: left;
}

.manual-label {
  font-size: 13px;
  font-weight: 600;
  color: #1e293b;
}

.manual-input-field {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 12px 14px;
  font-size: 13px;
  outline: none;
  font-family: var(--font-sans);
}

.manual-desc-mockup {
  font-size: 10px;
  color: #64748b;
  line-height: 1.4;
}

.manual-submit-btn {
  background-color: #194E9E;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 12px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.manual-tips {
  font-size: 9px;
  color: #64748b;
  line-height: 1.4;
}

.scanner-bottom-stats {
  position: absolute;
  bottom: 55px;
  left: 0;
  right: 0;
  z-index: 3;
  padding: 16px 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.scanner-fullscreen-container.manual-bg .scanner-bottom-stats {
  background: none;
}

.stats-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.stats-label {
  font-size: 13px;
  font-weight: 700;
  color: #ffffff;
  letter-spacing: 0.5px;
}

.scanner-fullscreen-container.manual-bg .stats-label {
  color: #64748b;
}

.stats-val {
  font-size: 13px;
  font-weight: 700;
  color: #ffffff;
}

.scanner-fullscreen-container.manual-bg .stats-val {
  color: #1e293b;
}

.stats-progress-container {
  width: 100%;
  height: 6px;
  background-color: rgba(255, 255, 255, 0.2);
  border-radius: 10px;
  overflow: hidden;
}

.scanner-fullscreen-container.manual-bg .stats-progress-container {
  background-color: #e2e8f0;
}

.stats-progress-fill {
  height: 100%;
  background-color: #194E9E;
  border-radius: 10px;
}

.ticket-category-bar {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 3;
  display: flex;
  align-items: center;
  background-color: #ffffff;
  padding: 8px 16px;
  box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.1);
}

.category-buttons-row {
  display: flex;
  width: 100%;
  gap: 10px;
}

.sheet-category-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: white;
  border: 1px solid #194E9E;
  color: #194E9E;
  padding: 8px 0;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.sheet-category-btn.active {
  border-color: #194E9E;
  background-color: #194E9E;
  color: white;
}

.scanner-fullscreen-container.manual-bg .ticket-category-bar {
  background-color: #194E9E;
}

.scanner-fullscreen-container.manual-bg .sheet-category-btn {
  background-color: #194E9E;
  border-color: rgba(255, 255, 255, 0.3);
  color: white;
}

.scanner-fullscreen-container.manual-bg .sheet-category-btn.active {
  background-color: white;
  color: #194E9E;
  border-color: white;
}

.notification-popup-card {
  position: absolute;
  bottom: 80px;
  left: 16px;
  right: 16px;
  background-color: white;
  border-radius: 12px;
  overflow: hidden;
  z-index: 100;
  color: #1e293b;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
}

.notification-header {
  padding: 10px 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.notification-header.success {
  background-color: #d1fae5;
  color: #065f46;
}

.notification-header.pending {
  background-color: #dbeafe;
  color: #1e40af;
}

.notification-header.failed {
  background-color: #fee2e2;
  color: #991b1b;
}

.notification-header.already {
  background-color: #fef3c7;
  color: #92400e;
}

.header-icon {
  display: flex;
  align-items: center;
}

.status-svg {
  width: 18px;
  height: 18px;
}

.header-text {
  font-size: 11px;
  font-weight: 700;
}

.notification-body {
  padding: 20px 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.attendee-row {
  display: flex;
  align-items: center;
  width: 100%;
  padding-bottom: 14px;
  border-bottom: 1px solid #e2e8f0;
}

.event-name {
  font-size: 12px;
  font-weight: 500;
  color: #1e293b;
  margin: 0 0 6px 0;
}

.attendee-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.invoice-text {
  font-size: 16px;
  font-weight: 700;
  margin: 0;
  color: #0f172a;
}

.buyer-name {
  font-size: 14px;
  font-weight: 500;
  color: #0f172a;
  margin: 0;
}

.ticket-type-label {
  font-size: 13px;
  font-weight: 500;
  color: #0f172a;
  margin: 0;
}

.scan-time-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.scan-time-info {
  display: flex;
  flex-direction: column;
}

.time-label {
  font-size: 9px;
  color: #64748b;
}

.time-val {
  font-size: 12px;
  font-weight: 700;
}

.continue-btn {
  background-color: #194E9E;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 6px 14px;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
}

.camera-error-card .error-card-desc {
  font-size: 11px;
  color: #475569;
  line-height: 1.5;
}

.error-action-row {
  display: flex;
  gap: 8px;
  width: 100%;
}

.manual-switch-btn {
  background-color: #e2e8f0;
  color: #475569;
}

/* Transitions */
.popup-slide-enter-active,
.popup-slide-leave-active {
  transition: transform 0.3s cubic-bezier(0.16, 1, 0.3, 1), opacity 0.2s ease;
}

.popup-slide-enter-from,
.popup-slide-leave-to {
  transform: translateY(120%);
  opacity: 0;
}
</style>
