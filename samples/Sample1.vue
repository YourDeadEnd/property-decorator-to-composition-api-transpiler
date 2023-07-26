<template>
  <div class="student-attendance">
    <m-preloader
      v-if="loading"
      position="absolute"
    />
    <div class="title">
      <svgicon name="calendar_2"/>
      {{ $t('student.goals.attendance.title') }}
    </div>
    <v-pie-card
      v-bind="pieCardProps"
      :text="$t('student.goals.attendance.title')"
      class="pie-card"
    />
    <attendance
      :items="items"
      :legend-data="legendData"
      :max-rows-amount="200"
      class="attendance"
    >
      <template #cards="renderedItems">
        <template v-for="card in renderedItems">
          <v-link
            :key="card.id"
            :to="card.to || {}"
            :disabled="!card.to"
            @click="sendEventAttendanceClick"
            @click.native.middle="sendEventAttendanceClick"
            @click.native.right="sendEventAttendanceClick"
          >
            <attendance-card
              :status="card.status"
              :icon="card.icon"
              :title="card.title"
              :date="card.date"
              class="card"
              :class="{ link: card.to }"
            >
              <template #icon>
                <svgicon
                  v-if="iconNameMap[card.status]"
                  :name="iconNameMap[card.status]"
                />
              </template>
            </attendance-card>
          </v-link>
        </template>
      </template>
    </attendance>
  </div>
</template>

<script lang="ts">
import { Vue, Component, Watch, Prop, VModel } from 'vue-property-decorator';
import { format } from 'date-fns';
import VPieCard from '@ui/VPieCard.vue';
import VLink from '@ui/VLink/VLink.vue';
import colors from '@modules/colors.module.scss';
import Attendance from '@/components/attendance/Attendance.vue';
import AttendanceCard from '@/components/attendance/AttendanceCard.vue';
import { store } from '@/store';
import { RouteName } from '@/enums/router';
import LessonTypeEnum from '@/enums/LessonTypeEnum';
import { ShowSolutionColors } from '@/store/modules/student/coursesGoals/types';
import LessonAttendanceTypeEnum from '@/enums/LessonAttendanceTypeEnum';
import ymm from '@/plugins/YandexMetrikaPlugin';
import type { StudentAttendanceResponse } from '@/store/modules/student/coursesGoals/types';
import type { CardWithId } from '@/components/attendance/types';
import type { Location } from 'vue-router';
import type { VLegendData } from '@ui/VLegend';

const coursesGoalsModule = store.student.modules.coursesGoals;

const GREEN = colors.green900;
const RED = colors.red200;
const BLUE = colors.mBlue700;
const GRAY = colors.neutral400;

enum StudentAttendanceCardStatuses {
  EMPTY = 'empty', // Пустая ячейка
  NULL = 'null', // Препод не проставил значение
  ONLINE = 'online', // Посещено
  REPLENISHMENT = 'replenishment', // Восполнение пропуска
  OFFLINE = 'offline', // Просмотрена запись
  MISSED = 'missed', // Пропущено
  HOLIDAYS = 'holidays', // Каникулы
}

type StudentAttendanceCard = CardWithId & {
  to?: Location;
};
type Items = StudentAttendanceCard[];

type PieCardProps = {
  percent?: number; // 0 - 100
  color?: ShowSolutionColors;
};

const visitedStatuses = [
  LessonAttendanceTypeEnum.ONLINE,
  LessonAttendanceTypeEnum.OFFLINE,
  LessonAttendanceTypeEnum.REPLENISHMENT,
];

@Component({
  components: {
    Attendance,
    AttendanceCard,
    VLink,
    VPieCard,
  },
})
export default class StudentAttendance extends Vue {
  loading = false;

  iconNameMap: Record<StudentAttendanceCardStatuses, string | undefined> = {
    [StudentAttendanceCardStatuses.EMPTY]: 'question',
    [StudentAttendanceCardStatuses.NULL]: undefined,
    [StudentAttendanceCardStatuses.ONLINE]: 'checked-box-2',
    [StudentAttendanceCardStatuses.REPLENISHMENT]: 'book-check',
    [StudentAttendanceCardStatuses.OFFLINE]: 'videocam-outline',
    [StudentAttendanceCardStatuses.MISSED]: 'cancel-outline',
    [StudentAttendanceCardStatuses.HOLIDAYS]: 'user-clock',
  };

  legendData: VLegendData = [
    {
      text: this.$t('student.goals.attendance.online'),
      icon: { name: 'checked-box-2', color: GREEN },
    },
    {
      text: this.$t('student.goals.attendance.replenishment'),
      icon: { name: 'book-check', color: GREEN },
    },
    {
      text: this.$t('student.goals.attendance.offline'),
      icon: { name: 'videocam-outline', color: GREEN },
    },
    {
      text: this.$t('student.goals.attendance.missed'),
      icon: { name: 'cancel-outline', color: RED },
    },
    {
      text: this.$t('student.goals.attendance.holidays'),
      icon: { name: 'user-clock', color: BLUE },
    },
    {
      text: this.$t('student.goals.attendance.empty'),
      icon: { name: 'question', color: GRAY, width: 12, height: 12 },
    },
  ];

  @VModel({ type: Boolean, required: true })
  show: boolean;

  @Prop({ type: Array, default: () => [] })
  testProp: any[];

  get currentStudentGroupId() {
    return store.root.state.currentStudentGroupId;
  }

  get items(): Items {
    if (!coursesGoalsModule.state.studentAttendance) return [];

    return coursesGoalsModule.state.studentAttendance.map((item) => {
      const { id, attendanceType, lessonType, startAtDate, webinarId, videoLink } = item;

      const to = webinarId && videoLink ? { name: RouteName.WEBINAR, params: { lessonId: `${id}` } } : undefined;
      let status = StudentAttendanceCardStatuses.NULL;

      if (lessonType === LessonTypeEnum.SELF_STUDY) {
        status = StudentAttendanceCardStatuses.HOLIDAYS;
      } else if (attendanceType) {
        status = attendanceType as unknown as StudentAttendanceCardStatuses;
      } else if (!startAtDate) {
        status = StudentAttendanceCardStatuses.EMPTY;
      }

      return {
        id,
        to,
        status,
        date: startAtDate ? format(startAtDate * 1000, 'dd.MM') : undefined,
      };
    });
  }

  get pieCardProps(): PieCardProps {
    const percent = this.getPercent(coursesGoalsModule.state.studentAttendance);
    if (percent === undefined) return {};

    const color: ShowSolutionColors = this.getColor(percent);

    return {
      percent,
      color,
    };
  }

  @Watch('currentStudentGroupId', { immediate: true })
  currentStudentGroupIdWatcher(value?: number) {
    if (!value) return;
    this.loadStudentAttendance(value);
  }

  async loadStudentAttendance(groupId: number) {
    if (this.loading) return;

    this.loading = true;
    const error = await coursesGoalsModule.dispatch('loadStudentAttendance', groupId);
    if (error) {
      this.$mNotify({
        title: this.$t('common.error'),
        text: 'Не удалось загрузить посещаемость',
      });
    }
    this.loading = false;
  }

  getPercent(studentAttendance: StudentAttendanceResponse | null): number | undefined {
    if (!studentAttendance || !studentAttendance.length) return undefined;

    const currentDate = Date.now();
    const allLessons = studentAttendance
      .filter(({ lessonType, startAtDate, attendanceType }) => {
        return lessonType !== LessonTypeEnum.SELF_STUDY
          && attendanceType
          && startAtDate
          && (startAtDate * 1000) < currentDate;
      });

      if (!allLessons.length) return undefined;

    const visitedLessons = allLessons.filter(({ attendanceType }) => visitedStatuses.includes(attendanceType!));

    return Math.round(visitedLessons.length / allLessons.length * 100);
  }

  getColor(percent: number): ShowSolutionColors {
    if (percent >= 0 && percent <= 60) return ShowSolutionColors.RED;
    if (percent >= 61 && percent <= 90) return ShowSolutionColors.YELLOW;
    return ShowSolutionColors.GREEN;
  }

  sendEventAttendanceClick() {
    const countMissed = this.items.filter(item => item.status === StudentAttendanceCardStatuses.MISSED).length;
    const titleEducationCourse: string | undefined = store.root.getters.currentGroup?.educationCourse.titleForModule;
    const crmUuid = store.auth.getters.authUser.crm_uuid;

    ymm.goalAndProgress.sendAttendanceClick({
      countMissed,
      titleEducationCourse,
      crmUuid,
    });
  }
}
</script>

<style lang="scss" scoped>
.student-attendance {
  position: relative;
}
.title {
  color: $simple-text;
  font-size: 17px;
  line-height: 20px;
  font-weight: 700;
  margin-bottom: 16px;
  display: flex;
  align-items: center;

  svg {
    margin-right: 7px;
  }
}

.pie-card {
  width: 172px;

  @include media-down(sm) {
    width: 100%;
    margin-bottom: 8px;
    margin-right: 0;
  }
}

.attendance .card {
  width: var(--card-width);
  height: var(--card-height);

  &.empty {
    border: 1px solid $m-neutral-400;

    svg {
      fill: $m-neutral-400;
    }
  }
  &.null {
    border: 1px solid $m-neutral-400;
  }
  &.online,
  &.offline,
  &.replenishment {
    background-color: $m-green-50;

    svg {
      fill: $green-900;
    }
  }
  &.missed {
    background-color: $red-50;

    svg {
      fill: $red-200;
    }
  }
  &.holidays {
    background-color: $m-blue-50;

    svg {
      fill: $m-blue-700;
    }
  }

  &.link {
    &:hover {
      opacity: 0.6;
    }

    &::v-deep .date {
      text-decoration: underline;
    }
  }
}
</style>
