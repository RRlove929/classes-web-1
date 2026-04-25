<!-- 课程评论列表 -->
<template>
  <div>
    <el-form :model="comment" class="comment_box">
      <div class="score_box">
        <div class="score_left">
          <div class="score_title">课程评分</div>
          <div class="score_avg">
            <el-rate :model-value="Number(scoreInfo.scoreAvg || 0)" disabled show-score allow-half text-color="#ff9900" />
            <span class="score_count">({{ scoreInfo.scoreCount || 0 }} 人评分)</span>
          </div>
        </div>
        <div class="score_right">
          <div class="score_title">{{ scoreInfo.myScore ? '我的评分' : '我要评分' }}</div>
          <el-rate
            v-model="rateValue"
            :disabled="rateDisabled"
            show-score
            text-color="#ff9900"
            score-template="{value}"
            @change="handleRateChange"
          />
          <div v-if="!hasToken" class="score_tip">登录后可评分（每门课仅一次）</div>
          <div v-else-if="scoreInfo.myScore" class="score_tip">已评分，不能重复评分</div>
        </div>
      </div>

      <el-input v-model="comment.commentText" :rows="5" class="comment-text" maxlength="200" show-word-limit type="textarea" placeholder="请输入您的评论..." />
      <el-button class="comment-button" type="primary" size="large" @click="handleComment"> 评论 </el-button>
    </el-form>
    <div class="course_comment_list clearfix">
      <div v-for="(comment, index) in page.list" :key="index" class="course_comment_item">
        <div class="course_comment_cover">
          <img :src="commonHead" class="comment_cover_img" />
        </div>
        <div class="course_comment_content">
          <div class="course_comment_user">
            匿名用户
          </div>
          <div class="course_comment_int">
            {{ comment.commentText }}
          </div>
          <div class="course_comment_btn">
            <div class="course_comment_time">
              {{ comment.gmtCreate }}
            </div>
          </div>
        </div>
      </div>
      <div v-if="props.showPage" class="pagination">
        <common-pagination v-model:current-page="page.pageCurrent" v-model:page-size="page.pageSize" :total="page.totalCount" @pagination="handlePage" />
      </div>
    </div>
  </div>
</template>

<script setup>
import useTable from '~/utils/table.js'
import { courseApi } from '~/api/course.js'
import { getToken } from '~/utils/cookie.js'
import commonHead from '~/assets/image/common_head.jpg'
const props = defineProps({
  courseId: {
    type: String,
    default: ''
  },
  /**
   * 是否匿名展示（头像/昵称）
   * true: 显示“匿名用户”+默认头像
   * false: 优先显示 usersVO
   */
  anonymous: {
    type: Boolean,
    default: true
  },
  showPage: {
    type: Boolean,
    default: true
  }
})

const comment = ref({
  courseId: props.courseId
})

const hasToken = ref(false)
const scoreInfo = ref({
  scoreAvg: null,
  scoreCount: 0,
  myScore: null
})
const rateValue = ref(0)
const rateDisabled = computed(() => {
  return !hasToken.value || !!scoreInfo.value.myScore
})

function getCommentHead(item) {
  if (props.anonymous) {
    return commonHead
  }
  return item?.usersVO?.userHead || commonHead
}

function getCommentName(item) {
  if (props.anonymous) {
    return '匿名用户'
  }
  return item?.usersVO?.nickname || '匿名用户'
}

async function loadScoreInfo() {
  hasToken.value = !!getToken()
  const detail = hasToken.value ? await courseApi.userCourseDetail({ courseId: props.courseId }) : await courseApi.courseDetail({ courseId: props.courseId })
  scoreInfo.value = {
    scoreAvg: detail?.scoreAvg ?? null,
    scoreCount: detail?.scoreCount ?? 0,
    myScore: detail?.myScore ?? null
  }
  rateValue.value = scoreInfo.value.myScore ? scoreInfo.value.myScore : 0
}

onMounted(() => {
  loadScoreInfo()
})

async function handleRateChange(val) {
  if (rateDisabled.value) {
    return
  }
  await courseApi.courseScoreAdd({ courseId: props.courseId, score: val })
  ElMessage.success('评分成功')
  await loadScoreInfo()
}

function handleComment() {
  if (!comment.value.commentText) {
    ElMessage.warning('请输入评论内容')
    return
  }
  courseApi.courseCommentAdd(comment.value).then((res) => {
    handlePage()
    comment.value.commentText = ''
  })
}
const { page, handlePage } = useTable(
  {
    page: courseApi.courseCommentPage
  },
  { courseId: props.courseId }
)
</script>

<style lang="scss" scoped>
.comment_box {
  padding: 20px;
  .score_box {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 14px;
    padding: 12px;
    background: #fafafa;
    border: 1px solid #ebeef5;
    border-radius: 6px;
    .score_title {
      color: #333;
      font-size: 14px;
      font-weight: 600;
      margin-bottom: 6px;
    }
    .score_avg {
      display: flex;
      align-items: center;
      gap: 8px;
    }
    .score_count {
      color: #999;
      font-size: 12px;
    }
    .score_tip {
      margin-top: 6px;
      color: #999;
      font-size: 12px;
    }
    .score_left {
      flex: 1;
      min-width: 260px;
    }
    .score_right {
      min-width: 260px;
      text-align: right;
    }
  }
  .comment-text {
    float: right;
    margin-bottom: 20px;
  }
  .comment-button {
    float: right;
  }
}
.course_comment_list {
  padding: 20px 20px 0 20px;
  .course_comment_item {
    border-bottom: 1px solid #ebeef5;
    display: flex;
    padding-bottom: 20px;
    padding-top: 20px;
  }
  .course_comment_cover {
    margin-right: 20px;
    .comment_cover_img {
      border-radius: 50%;
      width: 50px;
    }
  }

  .course_comment_content {
    width: calc(100% - 68px);
    .course_comment_int {
      color: #999;
      font-size: 14px;
      font-style: normal;
      font-weight: 400;
      margin-bottom: 10px;
      margin-top: 10px;
      word-wrap: break-word;
    }
  }
}
</style>