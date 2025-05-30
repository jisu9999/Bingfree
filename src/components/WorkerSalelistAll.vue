<template>
  <div class="tablelist">
    <h1 class="profile-h2">정산 현황</h1>
    <table class="table">
      <thead>
        <tr>
          <th>번호</th>
          <th>고객명</th>
          <th>정산번호</th>
          <th>정산유형</th>
          <th>연락처</th>
          <th>정산기간</th>
          <th>청소건수</th>
          <th>총 금액</th>
          <th>보너스</th>
          <th>수수료</th>
          <th>상태</th>
          <th>처리일자</th>
          <th>액션</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in paginatedList" :key="item.id">
          <td class="nowrap">{{ item.number }}</td>
          <td class="nowrap customername">
            <span v-html="item.customerIcon" style="margin-right: 4px"></span>
            {{ item.customerName }}
          </td>
          <td class="nowrap">{{ item.settlementId }}</td>
          <td class="nowrap">{{ item.type }}</td>
          <td class="nowrap">{{ item.contact }}</td>
          <td class="nowrap">{{ item.period }}</td>
          <td class="nowrap">{{ item.cleanCount }}</td>
          <td class="nowrap">{{ item.totalAmount.toLocaleString() }}원</td>
          <td class="nowrap">{{ item.bonus.toLocaleString() }}원</td>
          <td class="nowrap">{{ item.fee.toLocaleString() }}원</td>
          <td class="nowrap">
            <span :class="`statusbox-${item.status}`">
              {{ statusLabel(item.status) }}
            </span>
          </td>
          <td class="nowrap">{{ item.settledAt }}</td>
          <td class="btnbox nowrap">
            <button class="modal" @click="() => openModal(item.id)">처리하기</button>
            <button class="modal" @click="() => openDetailById(item.id)">상세보기</button>
          </td>
        </tr>
      </tbody>
    </table>

    <div class="pagination">
      <span>총 {{ filteredList.length }}건의 정산</span>
      <div class="pagebox">
        <button @click="goToPage(currentPage - 1)" :disabled="currentPage === 1">←</button>
        <button
          v-for="page in totalPages"
          :key="page"
          :class="{ active: currentPage === page }"
          @click="goToPage(page)">
          {{ page }}
        </button>
        <button @click="goToPage(currentPage + 1)" :disabled="currentPage === totalPages">→</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import { fullSaleList } from "@/data.mjs";

const icons = [
  `<svg
                    style="display: inline-block"
                    width="15"
                    height="15"
                    viewBox="0 0 15 15"
                    fill="none"
                    xmlns="http://www.w3.org/2000/svg"
                    xmlns:xlink="http://www.w3.org/1999/xlink">
                    <rect width="15" height="15" fill="url(#pattern0_273_889)" />
                    <defs>
                      <pattern id="pattern0_273_889" patternContentUnits="objectBoundingBox" width="1" height="1">
                        <use xlink:href="#image0_273_889" transform="scale(0.0185185)" />
                      </pattern>
                      <image
                        id="image0_273_889"
                        width="54"
                        height="54"
                        preserveAspectRatio="none"
                        xlink:href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADYAAAA2CAYAAACMRWrdAAAACXBIWXMAAAsSAAALEgHS3X78AAAEXUlEQVRogd1aPUwUQRR+u6HCxKNZO/QSO8BISGig4OproNAsNgKNJgYSGmjhEis0EaPBaAPRAi5SYHOUHgU2GIMJ0pkc0rmFnImUu+bbn9u9vdmfmb29H76EAHO7c++b7817b96uZBgGJYGi6lkiyiaaJBzHWlG+4L2Jm5ii6n1ENEtEU0Q00RTTo1ElojIR7eEnDtHYxGxC60Q0kzqNcFRtO9bDCMYipqj6rD1ZpmXmRwMEp7SiXGZdKUfdrqj6FhFtdhgpsu35bC96A0KJ2aTa7XpR2FRUfdF/TSAxRdVXu4CUgxd+5Zh7TFH1HGRusXFJgT03rBXlCoUott41dFxkvHY3ELMlvdse2xJjUlH1YQpQjBllugiLDcTs8ihRNTE+ING31zJ9WpGpX2nLaqAialAsJzobSIDM3opk/j02QPQgJ0Xe83PTWgQsCOtzjHMuUAbu6CfGXcxmrhG9emKpBDI8uKlIdL3XWgQsSH7UJbd835oT41ELxEADsWHeGZ7OSDQ9wf3FTAzZy4rfS/fcOU8q3FNl/cT6eGdIQqp6ya5TM731c1b/8c8dWSumCSgxVTDo72U9gf4b7peea0S/NP4zY087iQGHpwbllg1zT5WOLALnvy1COwcGrX0UOwj7ibXq4FgHkFjYcAmA7Mh8PSFn/8Xdby1RbDon0fgA0Z2sRIO33HG4IAyFUlDHv5cQcR/nreDkhPyFNwbtlKNVrCuCFVXn1l0rBm/TL6dWLoqbh96VDFrbdQki3LPuHZnXTZVDUKhZ5dRYzQTyE09yfZSXzGTtuN3D5zr9OHM/h8LPdt1gE4aaYqJHlTDFRAHDc8uuKnBHuOXbUqO7BqDQ9qjIAqqRD0uySQ4AIV54l5s7OacJBBkoJQqvYk3fY14gkHhdaXzQMhzqBAGfi6hFrQr3rBB9eEq0XTZCjzcYR2HsJG4epF5S7X8Nzjv+xMzCkGDz3EtM+CwWhh1mO9MFqgxvSPeDdU6Lg9QVg+FREKneo9DW6j5NeImlEhXjuFIavREvsVR681G5KKqnEceVWUjdFVEvTgf0LBDx3i+FExdoC5gwa0X72dcfkQni1opI0NsHhnmIBPKjFuGwBI2a8facLmJWrVZMteogW7kxztAtWnVQJ0dF5DbRtgB1KjG44PyGkAvW4BBretUR5zDIApSaLOjCQcNGJbFiKHBZQOGK1lpYueSFczpuAingOHF1jwL3pGI0NDnRC0SRi9YachWi4FBWqmuDgzSuKx0R7R/FPh1H4UwrysdNiYrWCgdvdCRZHFPCrmki8C5IbY911Ok5IcynmletCH7pfwZ9FRTDw/VV5x+HWLc+c/Yi530F6aq44hwioXegI/uKHID7zWpFec9/i2x3gLsRB/YLKw2kqEsVQy2zqhXlrbCLuoUY1MEe2vLvpSD0tOIs5kHVNtCBtzmHcSeqXcQlEISeBDnsu9eQMINF3ulNCscVIXXNkCCDg97m7DgQ0X+1TJk9rcZcSQAAAABJRU5ErkJggg==" />
                    </defs>
                  </svg>`,
  `<svg
                    style="display: inline-block"
                    width="14"
                    height="11"
                    viewBox="0 0 14 11"
                    fill="none"
                    xmlns="http://www.w3.org/2000/svg">
                    <path
                      d="M1.75 1.57143H0C0 4.60871 2.74258 7.07143 6.125 7.07143V10.6071C6.125 10.8232 6.32188 11 6.5625 11H7.4375C7.67812 11 7.875 10.8232 7.875 10.6071V7.07143C7.875 4.03415 5.13242 1.57143 1.75 1.57143ZM12.25 0C9.94766 0 7.94609 1.14174 6.89883 2.82857C7.65625 3.57009 8.2168 4.47121 8.51211 5.47054C11.5938 5.18326 14 2.84576 14 0H12.25Z"
                      fill="#4ECF50" />
                  </svg>`,
];

const settlements = fullSaleList.map((item) => {
  const randomIndex = Math.floor(Math.random() * icons.length);
  return {
    id: item.id,
    number: item.number,
    customerName: item.customer.name,
    customerIcon: icons[randomIndex],
    settlementId: item.customer.saleId,
    type: item.customer.type,
    contact: item.worker.mobile,
    period: item.customer.period,
    cleanCount: item.customer.cleannum,
    totalAmount: item.customer.totalPrice,
    bonus: item.customer.plusprice,
    fee: item.tex,
    settledAt: item.customer.settledAt,
    status: item.status,
    primemember: item.primemember,
  };
});

const currentPage = ref(1);
const itemsPerPage = 5;

const filteredList = computed(() => settlements);
const totalPages = computed(() => Math.ceil(filteredList.value.length / itemsPerPage));
const paginatedList = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage;
  return filteredList.value.slice(start, start + itemsPerPage);
});

const goToPage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
  }
};

const statusLabel = (status) => {
  switch (status) {
    case "waiting":
      return "대기중";
    case "assigned":
      return "진행중";
    case "done":
      return "청소완료";
    case "confirmed":
      return "확정완료";
    default:
      return "알수없음";
  }
};

const openModal = (id) => {
  console.log(`Modal open for ID: ${id}`);
};

const openDetailById = (id) => {
  console.log(`Open detail for ID: ${id}`);
};
</script>

<style scoped>
.nowrap {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 160px;
}
</style>
