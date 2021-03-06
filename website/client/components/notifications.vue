<template lang="pug">
div
  welcome-modal
  new-stuff
  death
  low-health
  level-up
  choose-class
  testing
  testingletiant
  rebirth-enabled
  drops-enabled
  contributor
  won-challenge
  ultimate-gear
  streak
  rebirth
  joined-guild
  joined-challenge
  invited-friend
</template>

<script>
// import moment from 'moment';

import { mapState } from 'client/libs/store';
import notifications from 'client/mixins/notifications';
import guide from 'client/mixins/guide';

import welcomeModal from './achievements/welcome';
import newStuff from './achievements/newStuff';
import death from './achievements/death';
import lowHealth from './achievements/lowHealth';
import levelUp from './achievements/levelUp';
import chooseClass from './achievements/chooseClass';
import armoireEmpty from './achievements/armoireEmpty';
import questCompleted from './achievements/questCompleted';
import questInvitation from './achievements/questInvitation';
import testing from './achievements/testing';
import testingletiant from './achievements/testingletiant';
import rebirthEnabled from './achievements/rebirthEnabled';
import dropsEnabled from './achievements/dropsEnabled';
import contributor from './achievements/contributor';
import invitedFriend from './achievements/invitedFriend';
import joinedChallenge from './achievements/joinedChallenge';
import joinedGuild from './achievements/joinedGuild';
import rebirth from './achievements/rebirth';
import streak from './achievements/streak';
import ultimateGear from './achievements/ultimateGear';
import wonChallenge from './achievements/wonChallenge';

export default {
  mixins: [notifications, guide],
  components: {
    wonChallenge,
    ultimateGear,
    streak,
    rebirth,
    joinedGuild,
    joinedChallenge,
    invitedFriend,
    welcomeModal,
    newStuff,
    death,
    lowHealth,
    levelUp,
    chooseClass,
    armoireEmpty,
    questCompleted,
    questInvitation,
    testing,
    testingletiant,
    rebirthEnabled,
    dropsEnabled,
    contributor,
  },
  data () {
    // Levels that already display modals and should not trigger generic Level Up
    let unlockLevels = {
      3: 'drop system',
      10: 'class system',
      50: 'Orb of Rebirth',
    };

    // Avoid showing the same notiication more than once
    let lastShownNotifications = [];
    let alreadyReadNotification = [];

    return {
      unlockLevels,
      lastShownNotifications,
      alreadyReadNotification,
      isRunningYesterdailies: false,
    };
  },
  computed: {
    ...mapState({user: 'user.data'}),
    // https://stackoverflow.com/questions/42133894/vue-js-how-to-properly-watch-for-nested-properties/42134176#42134176
    baileyShouldShow () {
      return this.user.flags.newStuff;
    },
    userHp () {
      return this.user.stats.hp;
    },
    userExp () {
      return this.user.stats.exp;
    },
    userGp () {
      return this.user.stats.gp;
    },
    userMp () {
      return this.user.stats.mp;
    },
    userLvl () {
      return this.user.stats.lvl;
    },
    userClassSelect () {
      return !this.user.flags.classSelected && this.user.stats.lvl >= 10;
    },
    userNotifications () {
      return this.user.notifications;
    },
    userAchievements () {
      // @TODO: does this watch deeply?
      return this.user.achievements;
    },
    armoireEmpty () {
      return this.user.flags.armoireEmpty;
    },
    questCompleted () {
      return this.user.party.quest.completed;
    },
    invitedToQuest () {
      return this.user.party.quest.RSVPNeeded && !this.user.party.quest.completed;
    },
  },
  watch: {
    baileyShouldShow () {
      this.$root.$emit('show:modal', 'new-stuff');
    },
    userHp (after, before) {
      if (after <= 0) {
        this.playSound('Death');
        this.$root.$emit('show:modal', 'death');
        // @TODO: {keyboard:false, backdrop:'static'}
      } else if (after <= 30 && !this.user.flags.warnedLowHealth) {
        this.$root.$emit('show:modal', 'low-health');
        // @TODO: {keyboard:false, backdrop:'static', controller:'UserCtrl', track:'Health Warning'}
      }
      if (after === before) return;
      if (this.user.stats.lvl === 0) return;
      this.hp(after - before, 'hp');

      // @TODO: I am pretty sure we no long need this with $store
      // this.$broadcast('syncPartyRequest', {
      //   type: 'user_update',
      //   user: this.user,
      // });

      if (after < 0) this.playSound('Minus_Habit');
    },
    userExp (after, before) {
      if (after === before) return;
      if (this.user.stats.lvl === 0) return;
      this.exp(after - before);
    },
    userGp (after, before) {
      if (after === before) return;
      if (this.user.stats.lvl === 0) return;

      let money = after - before;
      let bonus;
      if (this.user._tmp) {
        bonus = this.user._tmp.streakBonus || 0;
      }
      this.gp(money, bonus || 0);

      //  Append Bonus
      if (money > 0 && Boolean(bonus)) {
        if (bonus < 0.01) bonus = 0.01;
        this.text(`+ ${Notification.coins(bonus)} ${this.$t('streakCoins')}`);
        delete this.user._tmp.streakBonus;
      }
    },
    userMp (after, before) {
      if (after === before) return;
      if (!this.user.flags.classSelected || this.user.preferences.disableClasses) return;
      let mana = after - before;
      this.mp(mana);
    },
    userLvl (after, before) {
      if (after <= before) return;
      this.lvl();
      this.playSound('Level_Up');
      if (this.user._tmp && this.user._tmp.drop && this.user._tmp.drop.type === 'Quest') return;
      if (this.unlockLevels[`${after}`]) return;
      if (!this.user.preferences.suppressModals.levelUp) this.$root.$emit('show:modal', 'level-up');
    },
    userClassSelect (after) {
      if (!after) return;
      this.$root.$emit('show::modal', 'choose-class');
      // @TODO: {controller:'UserCtrl', keyboard:false, backdrop:'static'}
    },
    userNotifications (after) {
      if (!this.user._wrapped) return;
      if (this.user.needsCron) return;
      this.handleUserNotifications(after);
    },
    userAchievements () {
      this.playSound('Achievement_Unlocked');
    },
    armoireEmpty (after, before) {
      if (after === before || after === false) return;
      this.$root.$emit('show::modal', 'armoire-empty');
    },
    questCompleted (after) {
      if (!after) return;
      this.$root.$emit('show::modal', 'quest-completed');
    },
    invitedToQuest (after) {
      if (after !== true) return;
      this.$root.$emit('show::modal', 'quest-invitation');
    },
  },
  async mounted () {
    if (!this.user.flags.welcomed) {
      this.$root.$emit('show::modal', 'welcome');
    }

    Promise.all(['user.fetch', 'tasks.fetchUserTasks'])
      .then(() => {
        this.runYesterDailies();
      });

    window.setTimeout(() => {
      this.initTour();
      if (this.user.flags.tour.intro === this.TOUR_END) return;
      this.goto('intro', 0);
    }, 2000);
  },
  methods: {
    playSound () {
      // @TODO:
    },
    runYesterDailies () {
      // @TODO: Hopefully we don't need this even we load correctly
      if (this.isRunningYesterdailies) return;

      // let userLastCron = moment(this.user.lastCron).local();
      // let userDayStart = moment().startOf('day').add({ hours: this.user.preferences.dayStart });

      if (!this.user.needsCron) return;

      let dailys = this.$store.state.tasks.data.dailys;

      // @TODO: How do we check this now? if (!this.appLoaded) return;

      this.isRunningYesterdailies = true;

      // let yesterDay = moment().subtract('1', 'day').startOf('day').add({ hours: this.user.preferences.dayStart });
      let yesterDailies = [];
      dailys.forEach((task) => {
        if (task && task.group.approval && task.group.approval.requested) return;
        if (task.completed) return;
        // @TODO: let shouldDo = Shared.shouldDo(yesterDay, task);
        let shouldDo = false;

        if (task.yesterDaily && shouldDo) yesterDailies.push(task);
      });

      if (yesterDailies.length === 0) {
        // @TODO:
        // User.runCron().then(function () {
        //   isRunningYesterdailies = false;
        //   handleUserNotifications(this.user);
        // });
        return;
      }

      // @TODO:
      // let modalScope = this.$new();
      // modalScope.obj = this.user;
      // modalScope.taskList = yesterDailies;
      // modalScope.list = {
      //   showCompleted: false,
      //   type: 'daily',
      // };
      // modalScope.processingYesterdailies = true;
      //
      // $scope.yesterDailiesModalOpen = true;
      // $modal.open({
      //   templateUrl: 'modals/yesterDailies.html',
      //   scope: modalScope,
      //   backdrop: 'static',
      //   controller: ['$scope', 'Tasks', 'User', '$rootScope', function ($scope, Tasks, User, $rootScope) {
      //     this.$on('task:scored', function (event, data) {
      //       let task = data.task;
      //       let indexOfTask = _.findIndex($scope.taskList, function (taskInList) {
      //         return taskInList._id === task._id;
      //       });
      //       if (!$scope.taskList[indexOfTask]) return;
      //       $scope.taskList[indexOfTask].group.approval.requested = task.group.approval.requested;
      //       if ($scope.taskList[indexOfTask].group.approval.requested) return;
      //       $scope.taskList[indexOfTask].completed = task.completed;
      //     });
      //
      //     $scope.ageDailies = function () {
      //       User.runCron()
      //         .then(function () {
      //           isRunningYesterdailies = false;
      //           handleUserNotifications(this.user);
      //         });
      //     };
      //   }],
      // });
    },
    transferGroupNotification (notification) {
      if (!this.user.groupNotifications) this.user.groupNotifications = [];
      this.user.groupNotifications.push(notification);
    },
    handleUserNotifications (after) {
      if (!after || after.length === 0) return;

      let notificationsToRead = [];
      let scoreTaskNotification = [];

      this.user.groupNotifications = []; // Flush group notifictions

      after.forEach((notification) => {
        if (this.lastShownNotifications.indexOf(notification.id) !== -1) {
          return;
        }

        // Some notifications are not marked read here, so we need to fix this system
        // to handle notifications differently
        if (['GROUP_TASK_APPROVED', 'GROUP_TASK_APPROVAL'].indexOf(notification.type) === -1) {
          this.lastShownNotifications.push(notification.id);
          if (this.lastShownNotifications.length > 10) {
            this.lastShownNotifications.splice(0, 9);
          }
        }

        let markAsRead = true;
        // @TODO: Use factory function instead
        switch (notification.type) {
          case 'GUILD_PROMPT':
            // @TODO: I'm pretty sure we can find better names for these
            if (notification.data.textletiant === -1) {
              this.$root.$emit('show::modal', 'testing');
            } else {
              this.$root.$emit('show::modal', 'testingletiant');
            }
            break;
          case 'DROPS_ENABLED':
            this.$root.$emit('show::modal', 'drops-enabled');
            break;
          case 'REBIRTH_ENABLED':
            this.$root.$emit('show::modal', 'rebirth-enabled');
            break;
          case 'WON_CHALLENGE':
            this.$root.$emit('show::modal', 'won-challenge');
            break;
          case 'STREAK_ACHIEVEMENT':
            this.streak(this.user.achievements.streak);
            this.playSound('Achievement_Unlocked');
            if (!this.user.preferences.suppressModals.streak) {
              this.$root.$emit('show::modal', 'streak');
            }
            break;
          case 'ULTIMATE_GEAR_ACHIEVEMENT':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'ultimate-gear');
            break;
          case 'REBIRTH_ACHIEVEMENT':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'rebirth');
            break;
          case 'GUILD_JOINED_ACHIEVEMENT':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'joined-guild');
            break;
          case 'CHALLENGE_JOINED_ACHIEVEMENT':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'joined-challenge');
            break;
          case 'INVITED_FRIEND_ACHIEVEMENT':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'invited-friend');
            break;
          case 'NEW_CONTRIBUTOR_LEVEL':
            this.playSound('Achievement_Unlocked');
            this.$root.$emit('show::modal', 'contributor');
            break;
          case 'CRON':
            if (notification.data) {
              if (notification.data.hp) this.hp(notification.data.hp, 'hp');
              if (notification.data.mp) this.mp(notification.data.mp);
            }
            break;
          case 'GROUP_TASK_APPROVAL':
            this.transferGroupNotification(notification);
            markAsRead = false;
            break;
          case 'GROUP_TASK_APPROVED':
            this.transferGroupNotification(notification);
            markAsRead = false;
            break;
          case 'SCORED_TASK':
            // Search if it is a read notification
            for (let i = 0; i < this.alreadyReadNotification.length; i++) {
              if (this.alreadyReadNotification[i] === notification.id) {
                markAsRead = false; // Do not let it be read again
                break;
              }
            }

            // Only process the notification if it is an unread notification
            if (markAsRead) {
              scoreTaskNotification.push(notification);

              // Add to array of read notifications
              this.alreadyReadNotification.push(notification.id);
            }
            break;
          case 'LOGIN_INCENTIVE':
            //  @TODO: Notification.showLoginIncentive(this.user, notification.data, Social.loadWidgets);
            break;
          default:
            if (notification.data.headerText && notification.data.bodyText) {
              // @TODO:
              // let modalScope = this.$new();
              // modalScope.data = notification.data;
              // this.openModal('generic', {scope: modalScope});
            } else {
              markAsRead = false; // If the notification is not implemented, skip it
            }
            break;
        }

        if (markAsRead) notificationsToRead.push(notification.id);
      });

      let userReadNotifsPromise = false;
      // @TODO: User.readNotifications(notificationsToRead);

      if (userReadNotifsPromise) {
        userReadNotifsPromise.then(() => {
          // Only run this code for scoring approved tasks
          if (scoreTaskNotification.length > 0) {
            let approvedTasks = [];
            for (let i = 0; i < scoreTaskNotification.length; i++) {
              // Array with all approved tasks
              approvedTasks.push({
                params: {
                  task: scoreTaskNotification[i].data.scoreTask,
                  direction: 'up',
                },
              });

              // Show notification of task approved
              this.markdown(scoreTaskNotification[i].data.message);
            }

            // Score approved tasks
            // TODO: User.bulkScore(approvedTasks);
          }
        });
      }

      this.user.notifications = []; // reset the notifications
    },
    // @TODO: I think I have these handled in the http interceptor
    // this.$on('responseError500', function(ev, error){
    //   Notification.error(error);
    // });
    // this.$on('responseError', function(ev, error){
    //   Notification.error(error, true);
    // });
    // this.$on('responseText', function(ev, error){
    //   Notification.text(error);
    // });
  },
};
</script>
