<template>
    <div class="container">
        <div class="shadow"></div>
        <div class="status">
            <div class="status__img">
                <!-- <img :src="require(`img/arena/start-fight.png`)" alt=""> -->
            </div>
            <div class="status__text">
                {{title}}
            </div>
            <div v-if="isWin" class="status__reward-table reward-table">
                <div class="reward-table__bonus str">
                    <div class="reward-table__title">
                        Награда
                    </div>
                    <div class="reward-table__content">
                        <div
                            v-for="item in rewardItems"
                            :key="item.name"
                            class="reward-table__item"
                        >
                            <div class="img">
                                <img :src="require(`img/arena/${item.name}.png`)">
                            </div>
                            <div class="num">{{item.count}}</div>
                        </div>
                    </div>
                </div>
            </div>
            <div class="button" @click="goToMenu">Продолжить</div>
        </div>
    </div>
</template>
<script>
import expTable from 'js/expTable';

export default {
    name: 'FightStatus',
    props: {
        title: {
            type: String,
            default: ''
        },
        rewardItems: {
            type: Array,
            default: () => []
        }
    },
    data() {
        return {};
    },
    computed: {
        isWin() {
            return this.title === 'Победа!';
        },
        ...mapState('gameInfo', ['currentArenaInfo', 'selectedMissionId', 'activeTeam']),
        ...mapGetters('data', ['getMissions', 'getPersonages'])
    },
    methods: {
        goToMenu() {
            if (this.isWin) {
                this.addReward();
            }
            this.checkNewLevels(); // Проверка, мб кто то получил новый уровень
            this.setMissionProgress();
            this.changeLocation('MainMenu');
        },
        addReward() {
            this.rewardItems.forEach(item => {
                if (item.name === 'xp') { // засчитываем опыт всем персонажам юзера
                    this.addExperience({ activeTeam: this.activeTeam, xp: item.count });
                    return;
                }
                this.addGoods({
                    name: item.name,
                    count: item.count
                });
            });
        },
        setMissionProgress() {
            if (this.isWin) { // TODO: считать прогресс и при поражении по количеству выживших противников, но не перезаписывать, если уже была победа
                this.changeMissionProgress({
                    child: this.selectedMissionId,
                    id: this.currentArenaInfo.id,
                    progress: 100
                });
            }
        },
        checkNewLevels() {
            this.getPersonages.forEach(personage => {
                if (personage.attributes.xp >= expTable[personage.lvl]) {
                    personage.lvl += 1;
                    alert(personage.name + ' получил новый уровень ' + personage.lvl + '!');
                }
            });
        },
        ...mapMutations('gameInfo', ['changeLocation']),
        ...mapMutations('data', ['addGoods', 'changeMissionProgress', 'addExperience'])
    }
};
</script>
<style lang="scss" scoped>
.button {
  padding: 20px 30px;
  background: #eee;
  border-radius: 5px;
  cursor: pointer;
  position: relative;
}
.container {
  z-index: 99;
  position: absolute;
  width: 100%;
  height: 96%;
}
.shadow {
    position: fixed;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    background: rgba(0,0,0,.8);
}
.status {
  position: absolute;
  width: 100%;
  height: 80%;
  top: 10%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  &__img {
      position: absolute;
      top: 125px;
  }
  &__text {
    width: 665px;
    height: 218px;
    font-size: 64px;
    color: #fff;
    text-shadow: #474747 3px 5px 2px, 0px 0px 2px rgba(28,110,164,0);
    margin-bottom: 10px;
    position: relative;
    margin-bottom: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: url('~img/arena/start-fight.png') 0 20px no-repeat;
  }
  .reward-table {
    position: relative;
    display: flex;
    flex-direction: column;
    color: #fff;
    border-radius: 6px;
    border: 2px solid #fff;
    padding: 10px 20px;
    width: 70%;
    margin-bottom: 20px;
    .str {
      display: flex;
      border-bottom: 1px solid #fff;
      margin-bottom: 15px;
      padding-bottom: 10px;
      &:last-child {
        border-bottom: none;
        margin-bottom: 0;
      }
    }
    &__title {
      width: 30%;
      margin-bottom: 0;
    }
    &__content {
      width: 70%;
      display: flex;
      justify-content: center;
      // align-items: center;
    }
    &__item {
      display: flex;
      margin-right: 25px;
      .img {
        margin-right: 7px;
        img {
          width: 20px;
          height: 20px;
        }
      }
    }
  }
}
</style>
