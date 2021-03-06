<template>
    <div class="arena std-block">
        <Header></Header>
        <div
            class="arena__bg"
            :class="[`shake-bg--${shakeBgAnimation}`]"
        >
            <img src="~img/locations/location1.jpeg" alt="">
        </div>

        <FightBottom
            :personageId="currentActivePersonageId"
            :currentActiveSkill="currentActiveSkill"
            :allPersonages="allPersonages"
            @selectCurrentSkillActive="selectCurrentSkillActive($event)"
        />
        <FightStatus
            v-if="isFightStatusVisible"
            :title="fightStatusTitle"
            :rewardItems="currentArenaInfo.reward"
        />
        <transition name="slide">
            <FightMessage
                v-if="isMessageVisible"
                :currentSkillObj="currentSkillObj"
                :skillDamage="damageNum"
            />
        </transition>

        <div
            class="arena__field field"
            :class="[`shake-bg--${shakeBgAnimation}`]"
        >
            <div class="field__wrapper">
                <FightPersonage
                    v-for="personage in allPersonages"
                    :key="personage.id"
                    :personage="personage"
                    @attack="test"
                />
            </div>
        </div>

        <!-- <div class="test-btn" @click.once="startFight()">начать бой</div> -->
    </div>
    <!-- https://www.youtube.com/watch?v=y7Cq-0rXnB0 тут пример боевки -->
</template>

<script>
import { objectClone } from 'utils';
import Header from '../global/interface/Header';
import FightStatus from './interface/FightStatus';
import FightMessage from './interface/FightMessage';
import FightBottom from './interface/FightBottom';
import FightPersonage from './Personage/FightPersonage';
import getParam from 'js/getParam';

export default {
    name: 'Arena',
    components: {
        Header,
        FightStatus,
        FightMessage,
        FightBottom,
        FightPersonage
    },
    data() {
        return {
            fightTimer: '',
            isSelectEnemy: false,
            enemySelected: '',
            fightStatusTitle: '',
            isFightStatusVisible: false,
            shakeBgAnimation: '',
            isMessageVisible: false,
            damageNum: 0,
            currentSkillObj: {},

            currentActivePersonageId: '',
            currentActiveSkill: 0,

            enemies: this.personageGenerator('enemy'),
            team: this.personageGenerator('team'),
            allTimers: {
                mainInterval: 100,
                startDelay: 500, // задержка при mounted компонента
                enemyMoveDelay: 500, // задежка при каждом ходе соперника
                defaultAnimationDuration: 500, // если не указана продолжительность анимации скилла - будет это значение
                takeDamageAnimation: 1200, // запутался, что это
                damageAnimation: 300, // анимация получения повреждения
                switchingTeammetes: 500, // время анимация смены персонажей комманды
                shakeBgTimer: 300
            }
        };
    },
    computed: {
        allPersonages() {
            let array = [];
            array.push(...this.team);
            array.push(...this.enemies);
            return array;
        },
        ...mapState('gameInfo', ['activeTeam', 'currentArenaInfo'])
    },
    mounted() {
        setTimeout(() => {
            this.startFight();
        }, this.allTimers.startDelay);
    },
    beforeDestroy() {
        clearInterval(this.fightTimer);
    },
    methods: {
        selectCurrentSkillActive(num) {
            this.currentActiveSkill = num;
            // Если один соперник - сразу же его бьем
            if (this.enemies.length === 1) {
                this.isSelectEnemy = true;
            }
        },
        calculateDamage(personage) {
            console.dir(personage);
            let currentSkillName = personage.skills[this.currentActiveSkill].name;
            let skillInData = window.skills.find(skill => skill.name === currentSkillName);

            const getStr = getParam('strength', personage.stats.strength, personage.lvl);
            let damage = skillInData.damageCalc(getStr);
            personage.currentMana -= skillInData.manaCost;

            return damage;
        },
        getPersonagesByType(type) {
            if (type === 'enemy') {
                return this.$store.state.gameInfo.currentArenaInfo.enemies;
            }
            const { activeTeam } = this.$store.state.gameInfo;
            // Фильтруем по тем, кого выбрали в бой
            const userPersonages = this.$store.getters['data/getPersonages'].filter(personage => {
                if (activeTeam.includes(personage.id)) return personage;
                return false;
            });
            return userPersonages;
        },
        personageGenerator(type) {
            const personages = this.getPersonagesByType(type);

            return objectClone(personages).map((personage, index) => {
                personage.index = index;
                personage.type = type;

                personage.maxHP = getParam('stamina', personage.stats.stamina, personage.lvl);
                personage.currentHP = getParam('stamina', personage.stats.stamina, personage.lvl);

                personage.currentMana = getParam('intelligence', personage.stats.intelligence, personage.lvl);
                personage.maxMana = getParam('intelligence', personage.stats.intelligence, personage.lvl);

                personage.img = `personages/${personage.avatar}/ava1.png`;
                personage.position = 'default';
                personage.isNowDoingHit = false;
                personage.animationName = '';
                personage.damagedAnimation = false;
                personage.damagedNumAnimation = false;
                personage.currentDamage = 0;

                return personage;
            });
        },
        test(enemy) {
            let uncorrectSelect = this.team.find(el => el.id === enemy); // выбрал свою же команду
            if (uncorrectSelect) return;
            this.isSelectEnemy = true;
            this.enemySelected = enemy;
        },
        startFight(whosFirst) {
            let userArray = this.allPersonages.filter(personage => {
                return personage.type === 'team';
            });
            let computerArray = this.allPersonages.filter(personage => {
                return personage.type !== 'team';
            });

            // Вначале игры определяем, кто ходит первым
            if (!whosFirst) {
                whosFirst = this.chooseWhosFirstMove();
            }
            whosFirst === 'team' ? this.userMove(userArray, computerArray) : this.computerMove(computerArray, userArray);
        },
        chooseWhosFirstMove() {
            let maxPower = 0;
            let type = '';

            this.allPersonages.forEach(personage => {
                const getStr = getParam('strength', personage.stats.strength, personage.lvl);
                if (getStr > maxPower) {
                    maxPower = getStr;
                    type = personage.type;
                }
            });
            return type;
        },
        userMove(userArray, computerArray) {
            // по очереди ходит вся команда
            if (userArray.length > 0) {
                this.teamMemberMove(userArray, computerArray);
            } else {
                console.log('Вся команда походила');
                this.startFight('enemy');
            }
        },
        computerMove(computerArray, userArray) {
            if (computerArray.length > 0) {
                this.teamMemberMove(computerArray, userArray);
            } else {
                console.log('Все враги походили');
                this.startFight('team');
            }
        },
        teamMemberMove(team, otherTeam) {
            let isComputerTurn = team[0].type === 'enemy';

            // Активный прием сбрасываем
            this.currentActiveSkill = 0;

            // Выходит в центр
            this.moveToCenter(team[0]);

            if (isComputerTurn) {
                setTimeout(() => {
                    this.isSelectEnemy = true; // Не ждем хода человека
                }, this.allTimers.enemyMoveDelay);
            }
            if (this.enemies.length === 0) {
                return;
            }
            // Ждем, пока user нажмет на врага
            this.fightTimer = setInterval(() => {
                console.log('waiting for user touch');
                if (this.isSelectEnemy) {
                    // Сбрасываем данные о клике
                    this.isSelectEnemy = false;
                    clearInterval(this.fightTimer);
                    this.currentActivePersonageId = '';

                    if (otherTeam.length === 1) {
                        this.enemySelected = otherTeam[0].id;
                    }
                    // Выбор рандомного соперника, когда ходит противник
                    if (isComputerTurn) {
                        let opponentTeamLength = otherTeam.length;
                        let randomOpponent = Math.floor(Math.random() * opponentTeamLength);
                        this.enemySelected = otherTeam[randomOpponent].id;
                    }

                    // Создаем необходимые переменные
                    let currentPersonage = this.allPersonages.find(personage => {
                        return team[0].id === personage.id;
                    });
                    let currentEnemy = this.allPersonages.find(enemy => {
                        return this.enemySelected === enemy.id;
                    });
                    let currentSkillName = currentPersonage.skills[this.currentActiveSkill].name;
                    let skillInData = window.skills.find(skill => skill.name === currentSkillName);
                    this.currentSkillObj = skillInData;

                    // Подойти текущим персонажем к выбранному врагу
                    if (!skillInData.dontComeClose) { // есть скиллы, где не нужно подходить
                        currentPersonage.position = 'nearbyenemy';
                        currentPersonage.enemy = currentEnemy;
                    }

                    setTimeout(() => {
                        let calculateDamage = this.calculateDamage(currentPersonage);

                        // Анимация Удара
                        currentPersonage.isNowDoingHit = true;
                        currentPersonage.animationName = currentSkillName;
                        let animationDuration = skillInData.animationDuration || this.allTimers.defaultAnimationDuration;

                        // Анимация получения повреждений после анимации атаки
                        setTimeout(() => {
                            this.shakeBg(currentEnemy.type);
                            this.isMessageVisible = true;
                            currentPersonage.isNowDoingHit = false;
                            currentEnemy.damagedAnimation = true;
                            currentEnemy.currentDamage = calculateDamage;
                            this.damageNum = calculateDamage; // нужно будет избавиться
                            currentEnemy.damagedNumAnimation = true;
                        }, animationDuration);

                        setTimeout(() => {
                            currentEnemy.damagedNumAnimation = false;
                        }, animationDuration + this.allTimers.takeDamageAnimation);

                        // После двух анимаций продолжение
                        setTimeout(() => {
                            currentEnemy.damagedAnimation = false;

                            // Отнять хп
                            currentEnemy.currentHP -= calculateDamage;

                            // Если враг умер - вырезаем его из массива
                            if (currentEnemy.currentHP <= 0) {
                                currentEnemy.currentHP = 0;
                                console.log('враг умер');
                                let opponentTeamArray = isComputerTurn ? this.team : this.enemies;

                                opponentTeamArray.forEach((personage, index) => {
                                    if (personage.currentHP < 1) {
                                        opponentTeamArray.splice(index, 1);
                                    }
                                });
                                // Конец боя
                                if (opponentTeamArray.length === 0) {
                                    let titleMessage = team[0].type === 'team' ? 'Победа!' : 'Поражение...';
                                    this.fightStatusTitle = titleMessage;
                                    this.isFightStatusVisible = true;
                                }
                            }
                        }, animationDuration + this.allTimers.damageAnimation); // 300 мс длится анимация получения повреждений
                        setTimeout(() => {
                            this.isMessageVisible = false;
                            // Вернуться на default позицию
                            currentPersonage.enemy = false;
                            currentPersonage.position = 'default';

                            setTimeout(() => {
                                // ходит оставшаяся часть команды
                                let newArray = team.slice(1);
                                if (!isComputerTurn) {
                                    this.userMove(newArray, otherTeam);
                                } else {
                                    this.computerMove(newArray, otherTeam);
                                }
                            }, this.allTimers.switchingTeammetes);
                        }, animationDuration + this.allTimers.damageAnimation + this.allTimers.switchingTeammetes);
                    });
                }
            }, this.allTimers.mainInterval);
        },
        moveToCenter(personage) {
            personage.position = 'center';
            this.currentActivePersonageId = personage.id;
        },
        shakeBg(enemyType) {
            if (enemyType === 'enemy') {
                this.shakeBgAnimation = 'normal';
            } else {
                this.shakeBgAnimation = 'reverse';
            }
            setTimeout(() => {
                this.shakeBgAnimation = '';
            }, this.allTimers.shakeBgTimer);
        }
    }
};
</script>

<style lang="scss" scoped>
.field {
  position: absolute;
  width: 100%;
  height: calc(100% - 85px);
  bottom: 50px;
  &__wrapper {
    position: relative;
    width: 100%;
    height: 100%;
  }
}
.test-btn {
  position: fixed;
  width: 200px;
  height: 40px;
  cursor: pointer;
  left: 50%;
  top: 200px;
  background: #eee;
  text-align: center;
  line-height: 40px;
  border-radius: 6px;
}
.arena {
    &__bg {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      img {
        position: absolute;
        width: 100%;
        bottom: 0;
        left: 0;
      }
    }

  }
.shake-bg--normal {
    animation-name: shake-bg;
    animation-duration: .3s;
}
.shake-bg--reverse {
    animation-name: shake-bg-reverse;
    animation-duration: .3s;
}

@keyframes shake-bg {
    0%  {transform: translateX(0px);}
    33% {transform: translateX(30px);}
    66% {transform: translateX(-10px);}
    100% {transform: translateX(0px);}
}
@keyframes shake-bg-reverse {
    0%  {transform: translateX(0px);}
    33% {transform: translateX(-30px);}
    66% {transform: translateX(10px);}
    100% {transform: translateX(0px);}
}
.slide-enter-active, .slide-leave-active {
    transition: .5s;
}
.slide-enter {
    opacity: 0;
    transform: translateY(-100px);
}
</style>
