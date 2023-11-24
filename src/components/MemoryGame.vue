<template>

  <div class="memory-game">
    <div v-if="isLoading" class="loading-screen">
      加载中，请稍候...
      <!-- 可以添加一些动画或者进度条 -->
    </div>
    <!--时间展示区-->
    <div class="timer">
      <img :src="timerIcon" alt="Time Left"/>
      <div style="font-weight:bolder;">{{ timeLeft.toFixed(2) }}</div>
    </div>

    <!--分数展示区-->
    <div class="score">
      <img :src="coinIcon" alt="Time Left"/>
      <div style="font-weight:bolder;"> {{ score }}</div>
    </div>
    <div class="lives">
      <img v-for="n in lives" :key="n" :src="heartIcon" alt="Life"/>
    </div>
    <!--    游戏记忆区-->
    <div class="reminder" v-if="showRemenberImage">请记住宝可梦</div>
    <div class="images-to-remember" :class="{ hide: hideMemoryImage }" v-if="showRemenberImage">
      <img :src="currentImage" alt="宝可梦"/>
      <div class="image-info">
        <div class="imageName">{{ currentImageName }}</div>
        <div class="type-badges">
          <div v-for="type in parsedTypes" :class="['type-badge', type.class]" :key="type.class">
            {{ type.name }}
          </div>
        </div>
      </div>
    </div>

    <!--游戏选择区-->
    <div class="choices" v-if="showChoices">
      <div class="prompt">请选择上一只宝可梦</div>
      <div class="images-grid">
        <div v-for="(image, index) in imagesToShow" :key="index" :class="{ hide: image.hide }"
             @click="selectImage(image)">
          <div class="choice-image">

            <img :src="image.src" :alt="`Choice ${index + 1}`" :class="getTypeClass(image.typeNames)"/>
            <div v-if="showImageInfo" class="image-info">
              <div v-if="showImageName">{{ image.name }}</div>
              <div class="type-badges">
                <div v-for="type in parseTypes(image.typeNames)" :class="['type-badge', type.class]" :key="type.class">
                  {{ type.name }}
                </div>
              </div>
            </div>
            <img v-if="image.feedbackIcon" :src="image.feedbackIcon" class="feedback-icon"/>
          </div>
        </div>
      </div>
    </div>
    <!--    游戏菜单-->
    <div class="game-menu-popup" v-if="isGameOver">
      <div class="popup-content">
        <h2>{{ gameTitle }}</h2>
        <h3 style="color: #40ad43">{{ playTime }}秒内答对{{accessScore}}分即可通关</h3>
        <h3 style="color: #da4a2b">答对1题加{{addTime}}分</h3>

        <div class="broken-heart" v-if="showBrokenHeart">
          <img :src="brokenHeartIcon" alt="Broken Heart"/>
        </div>
        <div class="broken-heart" v-if="showGoodHeart">
          <img :src="heartIcon" alt="Heart"/>
        </div>
        <p>得分 {{ score }}</p>
        <p>回合数 {{ round }}</p>
        <button @click="startGame()" class="restart">{{ startButtonText }}</button>
      </div>
    </div>


  </div>

</template>

<script>
// 导入 JSON 数据
import pokemonData from '@/assets/data.json';
import correctSound from '@/assets/audio/correct.mp3';
import wrongSound from '@/assets/audio/wrong.mp3';
import backgroundSound from '@/assets/audio/bgm.mp3';
import gameOverSound from '@/assets/audio/gameover.mp3';
import clickSound from '@/assets/audio/click.mp3';
import trueIconSrc from '@/assets/image/true.svg';
import falseIconSrc from '@/assets/image/false.svg';
import shalouIconSrc from '@/assets/image/shalou.svg';
import coinIconSrc from '@/assets/image/coin.svg';
import heartIconSrc from '@/assets/image/heart.svg';
import brokenHeartSrc from '@/assets/image/broken-heart.gif';

export default {
  name: "MemoryGame",
  data() {
    return {
      showRemenberImage: false,
      maxPokemonIndex: 1000,
      gameTitle: '宝可梦记忆大师',
      startButtonText: '开始游戏',
      showChoices: false,
      showImageName: false,
      isLoading: true,
      showImageInfo: false,
      hideMemoryImage: false, // 新增属性，用于控制记忆图片的隐藏
      showBrokenHeart: false, // 控制心碎动态图的显示
      showGoodHeart: false, // 控制心动动态图的显示
      allImages: [],
      currentImageIndex: 0,
      currentImage: '',
      currentImageName: '',
      currentTypeName: '',
      isGameOver: false,
      imagesToShow: [],
      previousImage: '',
      previousTypeName: '',
      round: 0,//默认关卡
      timeLeft: 0.00,
      playTime: 30.00,
      lives: 3,
      score: 0,
      accessScore: 30,//通关分数
      volume: 0.1, // 默认音量
      addTime:0.5,//答对额外加秒数 s
      timer: null,
      errorTime: 0,
      maxErrorTime: 3,
      timerIcon: shalouIconSrc,
      coinIcon: coinIconSrc,
      heartIcon: heartIconSrc,
      brokenHeartIcon: brokenHeartSrc,
      correctSound: new Audio(correctSound),
      wrongSound: new Audio(wrongSound),
      backgroundSound: new Audio(backgroundSound),
      clickSound: new Audio(clickSound),
      gameOverSound: new Audio(gameOverSound),
      typeMapping: {
        'normal': '一般',
        'fire': '火',
        'water': '水',
        'electric': '电',
        'grass': '草',
        'ice': '冰',
        'fighting': '格斗',
        'poison': '毒',
        'ground': '地面',
        'flying': '飞行',
        'psychic': '超能力',
        'bug': '虫',
        'rock': '岩石',
        'ghost': '幽灵',
        'dragon': '龙',
        'dark': '恶',
        'steel': '钢',
        'fairy': '妖精',

      },

    };
  },
  computed: {
    parsedTypes() {
      return this.currentTypeName.split(',').map(type => ({
        class: type.trim().toLowerCase(),
        name: this.typeMapping[type.trim().toLowerCase()] || type.trim()
      }));
    },
  },
  created() {

    this.loadImages();
    this.isGameOver = true;
  },
  methods: {
    startRound() {
      const roundImageUrls = this.getCurrentRoundImageUrls();
      this.preloadImages(roundImageUrls).then(() => {
        // 图片加载完成，可以开始或继续回合
        // 更新游戏状态，显示回合内容等
      });
    },
    getCurrentRoundImageUrls() {
      // 基于当前回合的逻辑返回需要的图片 URL 数组
      // 这里是一个示例，具体逻辑可能需要根据您的游戏设计调整
      const roundImages = this.allImages.slice(0, this.currentRoundImageCount);
      return roundImages.map(image => image.src);
    },
    loadInitialRoundImages() {
      const initialRoundImageUrls = this.getCurrentRoundImageUrls(); // 获取第一轮的图片 URL
      return this.preloadImages(initialRoundImageUrls);
    },
    startGame() {

      console.log("starting game");
      this.loadInitialRoundImages().then(() => {
        // 图片加载完成后再开始游戏
        this.isLoading = false;
        this.playClickSound();
        console.log("starting game");
        this.isGameOver = false;
        this.initialShow(); // 初始化显示
        this.timeLeft = this.playTime;
        this.score = 0;
        this.round = 1;
        this.lives = 5;
        this.errorTime = 0;
        this.currentImageIndex = 0;
        this.stopGameOverSound();
        this.playBgmSound();
      });
    },
    endGame() {
      clearInterval(this.timer);
      this.isGameOver = true;
      if (this.score < 10) {
        this.gameTitle = '游戏结束,你是笨蛋，建议重开';
        this.showBrokenHeart = true;
        this.showGoodHeart = false;
        this.playGameOverSound()
      } else if (this.score >= 10 && this.score < 15) {
        this.gameTitle = '闯关失败,您的记忆水平为：一般';
        this.showBrokenHeart = true;
        this.showGoodHeart = false;
        this.playGameOverSound()
      } else if (this.score < this.accessScore) {
        this.gameTitle = '闯关失败,您的记忆水平为：中等';
        this.showBrokenHeart = true;
        this.showGoodHeart = false;
        this.playGameOverSound()
      } else if (this.score >= this.accessScore) {
        this.gameTitle = '闯关成功,您的记忆水平为：顶级';
        this.showBrokenHeart = true;
        this.showGoodHeart = false;
        this.playGameOverSound()
      }

      this.startButtonText = '重新开始';

      this.showChoices = false;
      this.showRemenberImage = false;
      this.errorTime = 0; // 重置错误次数
      this.stopBackgroundSound();

    },
    parseTypes(typeNames) {
      return typeNames.split(',').map(type => ({
        class: type.trim().toLowerCase(),
        name: this.typeMapping[type.trim().toLowerCase()] || type.trim()
      }));
    },
    getTypeClass(typeNames) {
      const firstType = typeNames.split(',')[0].trim().toLowerCase();
      return `border-${firstType}`;
    },
    // 预加载指定的图片
    preloadImages(imageUrls) {
      let loadedCount = 0;
      const totalImages = imageUrls.length;

      return new Promise((resolve) => {
        imageUrls.forEach(url => {
          const img = new Image();
          img.onload = img.onerror = () => {
            loadedCount++;
            if (loadedCount === totalImages) {
              resolve(); // 当所有指定图片加载完成时，解决 Promise
            }
          };
          img.src = url;
        });
      });
    },
    loadImages() {
      const baseUrl = 'https://www.pokemon.cn/play/resources/pokedex';
      this.allImages = pokemonData.pokemons
          .filter(pokemon => pokemon.zukan_sub_id !== 1) // 过滤掉 zukan_sub_id 为 1 的宝可梦
          .slice(0, this.maxPokemonIndex) // 限制数组大小为前 151 个
          .map(pokemon => ({
            id: pokemon.zukan_id,
            src: `${baseUrl}${pokemon.file_name}`,
            name: pokemon.pokemon_name,
            typeNames: pokemon.pokemon_type_id // 假设这是属性类型的字段
          }));
    },
    startTimer() {
      clearInterval(this.timer);
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft = parseFloat((this.timeLeft - 0.01).toFixed(2));
        } else {
          this.timeLeft = 0;
          this.endGame();
        }
      }, 10);
    },
    playBgmSound() {
      this.backgroundSound.volume = this.volume
      this.backgroundSound.play();
    },
    stopGameOverSound() {
      this.gameOverSound.pause();
      this.gameOverSound.currentTime = 0; // 重置音频到开始位置
    },
    stopBackgroundSound() {
      this.backgroundSound.pause();
      this.backgroundSound.currentTime = 0; // 重置音频到开始位置
    },
    playIsCorrectSound(isCorrect) {
      const sound = isCorrect ? this.correctSound : this.wrongSound;
      sound.play();
    },
    stopPlayIsCorrectSound(isCorrect) {
      const sound = isCorrect ? this.correctSound : this.wrongSound;
      sound.pause();
      sound.currentTime = 0; // 重置音频到开始位置
    },
    playClickSound() {
      // 假设你已经有了点击音效的音频文件
      this.clickSound.play();
    },
    playGameOverSound() {
      this.gameOverSound.play();
    },
    initialShow() {
      this.currentImageIndex = Math.floor(Math.random() * this.allImages.length);
      this.currentImage = this.allImages[this.currentImageIndex].src;
      this.currentImageName = this.allImages[this.currentImageIndex].name;
      this.currentTypeName = this.allImages[this.currentImageIndex].typeNames;
      this.showRemenberImage = true;
      setTimeout(() => {
        this.showRemenberImage = true;
        this.showChoices = true;
        this.startTimer();
        this.setupGame();
      }, 3000);
    },
    setupGame() {
      // 设置前一张图片和下一张要记忆的图片
      this.previousImage = this.currentImage;
      this.previousTypeName = this.currentTypeName;
      this.currentImageIndex = Math.floor(Math.random() * this.allImages.length);
      this.currentImage = this.allImages[this.currentImageIndex].src;
      this.currentImageName = this.allImages[this.currentImageIndex].name;
      this.currentTypeName = this.allImages[this.currentImageIndex].typeNames;

      // 动态计算要显示的图片数量
      let numberOfImagesToShow;
      if (this.round <= 2) {
        numberOfImagesToShow = 20;
      } else if (this.round <= 4) {
        numberOfImagesToShow = 20;
      } else if (this.round <= 6) {
        numberOfImagesToShow = 20;
      } else if (this.round <= 9) {
        numberOfImagesToShow = 20;
      } else {
        numberOfImagesToShow = 20; // 从第7回合开始，始终显示8张图片
      }

      // 获取随机图片列表
      this.imagesToShow = this.getRandomImages(this.previousImage, numberOfImagesToShow);
    },


    getRandomImages(previousImage, numberOfImages) {
      let currentType = this.previousTypeName.split(',')[0];
      let images = this.allImages
          .filter(image => image.src !== previousImage)
          .filter(image => image.typeNames.split(',')[0] === currentType)
          .sort(() => 0.5 - Math.random())
          .slice(0, numberOfImages - 1);
      images.push(this.allImages.find(image => image.src === previousImage));
      return images.sort(() => 0.5 - Math.random());
    },
    selectImage(selectedImage) {
      this.hideMemoryImage = true;
      const isCorrect = selectedImage.src === this.previousImage;
      this.score += isCorrect ? 1 : -1;
      if (!isCorrect) {
        this.errorTime += 1;
        this.timeLeft -= this.addTime;
        this.lives -= 1; // 减少一个生命值
        if (this.lives <= 0) {
          this.endGame(); // 如果生命值耗尽，则结束游戏
        }

      } else {
        this.timeLeft += this.addTime;
      }

      this.playIsCorrectSound(isCorrect);
      this.showImageName = true;
      this.imagesToShow = this.imagesToShow.map(image => ({
        ...image,
        feedbackIcon: image.src === selectedImage.src ? (isCorrect ? trueIconSrc : falseIconSrc) : '',
        hide: image.src !== selectedImage.src
      }));

      // 延迟1秒后继续下一回合
      setTimeout(() => {
        this.round++;
        this.setupGame();
        this.hideMemoryImage = false; // 显示记忆图片
        this.showImageName = false;// 不显示图片名字
      }, 300);

    }
  }
};
</script>
<style scoped>
.memory-game {
  padding: 10px;
  position: relative;
  max-width: 100%; /* 确保最大宽度不超过屏幕宽度 */
  width: 100%; /* 改为100%以适应屏幕宽度 */
  min-height: 95vh; /* 设置最小高度为视窗高度 */
  margin: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: #f7f7f7;
  border-radius: 15px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  overflow: hidden; /* 防止内容溢出容器 */
}
.loading-screen {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
  text-align: center;
  font-size: 20px;
  color: #333;
}
.image-info {
  display: flex;
  align-items: center; /* 垂直居中对齐 */
  justify-content: space-around; /* 两个元素之间有一些空间 */
  width: 100%; /* 可调整宽度以适应你的布局需求 */
}

.images-grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.images-grid img {
  width: 400px; /* 调整图片大小 */
  height: 400px;
  margin: 3px;
  object-fit: cover;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1); /* 可选：添加轻微的阴影效果 */

}

.images-to-remember {
  display: flex;
  flex-direction: column; /* 设置为垂直布局 */
  align-items: center; /* 水平居中对齐 */
  justify-content: center; /* 垂直居中对齐 */
  margin-bottom: 20px; /* 添加一些底部边距 */
  animation: slideIn 0.5s ease-out;
}

.images-to-remember img {
  width: 400px; /* 调整图片大小 */
  height: 400px;
  margin-bottom: 10px; /* 在图片和文字之间添加一些间距 */

}

.images-to-remember img.hide {
  visibility: hidden; /* 隐藏图片但保持其占据的空间 */
  opacity: 0; /* 使图片完全透明 */
  transition: opacity 0.5s; /* 可选：添加渐变效果 */
}

.imageName {
  text-align: center; /* 文字居中 */
  font-size: 18px; /* 设置文字大小 */
  color: #333; /* 设置文字颜色，可根据需要调整 */
}

.choices {
  width: 90%;
}

.choice-image {
  cursor: pointer;
  margin: 2px;
  /*transition: transform 0.3s ease, box-shadow 0.3s ease; !* 添加 transform 和 box-shadow 的过渡效果 *!*/
  position: relative;
  flex-basis: calc(33% - 10px); /* 调整为每行两张图片 */
  transition: border-color 0.9s; /* 平滑过渡效果 */
  animation: slideIn 0.5s ease-out;

}

@keyframes slideIn {
  from {
    margin: -100px;
  }
  to {
    margin: 2px;
  }
}

.choice-image.hide {
  opacity: 0; /* 使图片完全透明 */
  visibility: hidden; /* 隐藏图片但保持其占据的空间 */
  transition: opacity 0.5s; /* 可选：添加渐变效果 */
}

.volume-control {
  position: absolute;
  top: 10px;
  left: 50px;
}

.game-menu-popup {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center; /* 确保内部内容居中 */
}

.popup-content {
  background-color: white;
  border-radius: 10px;
  text-align: center;
  padding: 30px; /* 增加一些内部空间 */
  width: 60%;
}
@media (min-width: 375px) {
  .timer, .score {
    top: 5px;
    font-size: 14px;
  }

  .timer {
    right: 5px;
  }

  .timer img, .score img {
    width: 20px;
    height: 20px;
  }

  .score {
    left: 5px;
  }

  .score {
    top: 5px;
    left: 5px;
    font-size: 16px;
  }

  .score img {
    width: 25px;
    height: 25px;
  }

  .choice-image.hide {
    opacity: 0; /* 使图片完全透明 */
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    transition: opacity 0.5s; /* 可选：添加渐变效果 */
  }
  .choices {
    width: 100%;
  }

}
/* 媒体查询，适应大屏幕 */
@media (min-width: 376px) {
  .images-to-remember img,
  .images-grid img {
    width: 80px; /* 减小图片大小 */
    height: 80px;
  }
  .choices {
    width: 100%;
  }
  .choice-image {
    flex-basis: calc(33% - 10px); /* 每排三张图片，减去边距 */
    margin: 2px;
  }

  .images-to-remember img.hide {
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    opacity: 0; /* 使图片完全透明 */
    transition: opacity 1s; /* 可选：添加渐变效果 */
  }


  .choice-image.hide {
    opacity: 0; /* 使图片完全透明 */
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    transition: opacity 0.5s; /* 可选：添加渐变效果 */
  }

  .choice-image:nth-child(n+5) {
    flex-basis: calc(33% - 10px); /* 调整为每行两张图片 */
  }

  .choice-image:nth-child(n+7) {
    flex-basis: calc(12.5% - 10px); /* 每行八张图片 */
  }

  .imageName {
    font-size: 18px; /* 在小屏幕上减小文字大小 */
    font-weight: bolder;
  }
}

/* 媒体查询，适应大屏幕 */
@media (min-width: 390px) {
  .images-to-remember img,
  .images-grid img {
    width: 79px; /* 减小图片大小 */
    height: 79px;
  }
  .choices {
    width: 100%;
  }
  .choice-image {
    flex-basis: calc(33% - 10px); /* 调整为每行两张图片 */
    margin: 0.1px;
  }

  .images-to-remember img.hide {
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    opacity: 0; /* 使图片完全透明 */
    transition: opacity 1s; /* 可选：添加渐变效果 */
  }


  .choice-image.hide {
    opacity: 0; /* 使图片完全透明 */
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    transition: opacity 0.5s; /* 可选：添加渐变效果 */
  }

  .choice-image:nth-child(n+5) {
    flex-basis: calc(25% - 10px); /* 每行五张图片 */
  }

  .choice-image:nth-child(n+7) {
    flex-basis: calc(12.5% - 10px); /* 每行八张图片 */
  }

  .imageName {
    font-size: 14px; /* 在小屏幕上减小文字大小 */
    font-weight: bolder;
  }
}
/* 媒体查询，适应大屏幕 */
@media (min-width: 414px) {
  .images-to-remember img,
  .images-grid img {
    width:85px; /* 减小图片大小 */
    height: 85px;
  }
  .choices {
    width: 100%;
  }
  .choice-image {
    flex-basis: calc(20% - 2px); /* 每排三张图片，减去边距 */
    margin: 0.1px;
  }

  .images-to-remember img.hide {
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    opacity: 0; /* 使图片完全透明 */
    transition: opacity 3s; /* 可选：添加渐变效果 */
  }


  .choice-image.hide {
    opacity: 0; /* 使图片完全透明 */
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    transition: opacity 0.5s; /* 可选：添加渐变效果 */
  }

  .choice-image:nth-child(n+5) {
    flex-basis: calc(25% - 10px); /* 每行五张图片 */
  }

  .choice-image:nth-child(n+7) {
    flex-basis: calc(12.5% - 10px); /* 每行八张图片 */
  }

  .imageName {
    font-size: 14px; /* 在小屏幕上减小文字大小 */
    font-weight: bolder;
  }
}
@media (min-width: 768px) {
  /* 手机端的断点，可以根据需要调整 */
  .choice-image {
    flex-basis: calc(33% - 10px); /* 每排三张图片，减去边距 */
    margin: 3px;
  }
  .images-grid img {
    width: 108px; /* 减小图片大小 */
    height: 108px;
  }
  .images-to-remember img {
    width: 108px; /* 调整图片大小 */
    height: 108px;
    margin-bottom: 1px; /* 在图片和文字之间添加一些间距 */

  }
  .choice-image.hide {
    opacity: 0; /* 使图片完全透明 */
    visibility: hidden; /* 隐藏图片但保持其占据的空间 */
    transition: opacity 0.5s; /* 可选：添加渐变效果 */
  }
  .choices {
    width: 80%;
  }
}


.feedback-icon {
  position: absolute; /* 使图标浮动在图片上方 */
  top: 50%; /* 垂直居中 */
  left: 50%; /* 水平居中 */
  transform: translate(-50%, -50%); /* 确保图标完全居中 */
  width: 50px; /* 设定图标大小 */
  height: 50px;
  border-radius: 50%; /* 圆形图标 */
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1); /* 可选：添加轻微的阴影效果 */
}

.prompt {
  text-align: center; /* 居中文本 */
  width: 100%; /* 确保宽度充满容器 */
  margin-bottom: 1px; /* 添加一些下边距 */
}


/* 通用样式，适用于所有屏幕尺寸 */
.timer {
  position: absolute;
  top: 10px;
  right: 10px;
  display: flex;
  align-items: center;
  font-size: 18px;
  margin-right: 10px;

}

.timer img {
  width: 30px;
  height: 30px;
  margin-right: 10px;
}

/* 在屏幕左上角添加得分显示 */
.score {
  position: absolute;
  top: 10px;
  left: 10px;
  display: flex;
  align-items: center;
  font-size: 18px;
}

.score img {
  width: 30px;
  height: 30px;
  margin-right: 10px;
}

.type-badge {
  display: inline-block;
  padding: 5px 10px;
  margin: 2px;
  border-radius: 10px;
  color: white;
  font-size: 12px;
  font-weight: bolder;
  text-align: center;
}

/* 游戏结束弹窗内的按钮基本样式 */
.game-over-popup button {
  padding: 10px 20px;
  font-size: 16px;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  outline: none;
  margin: 10px;
  transition: all 0.3s ease;
}

/* 重新开始按钮样式 */
.game-over-popup button.restart {
  background-color: #4CAF50; /* 绿色背景 */
}

/* 重新开始按钮悬停效果 */
.game-over-popup button.restart:hover {
  background-color: #45a049; /* 深绿色背景 */
}

/* 关闭按钮样式 */
.game-over-popup button.close {
  background-color: #f44336; /* 红色背景 */
}

/* 关闭按钮悬停效果 */
.game-over-popup button.close:hover {
  background-color: #d32f2f; /* 深红色背景 */
}

.broken-heart img {
  width: 80px; /* 根据需要调整大小 */
  height: auto; /* 自动调整高度以保持图像比例 */
  margin: 10px 0; /* 在图标上下添加一些间隙 */
}

.popup-content h2 {
  font-size: 24px; /* 标题字体大小 */
  color: #333; /* 标题颜色 */
  margin-bottom: 20px; /* 标题与内容的间距 */
}

.popup-content p {
  font-size: 18px; /* 段落字体大小 */
  color: #555; /* 段落颜色 */
  margin: 10px 0; /* 段落间距 */
}

.popup-content .score, .popup-content .round {
  font-weight: bold; /* 字体加粗 */
  color: #4a90e2; /* 字体颜色 */
}

/* 按钮样式 */
.popup-content button {
  padding: 10px 20px;
  font-size: 16px;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  outline: none;
  margin: 10px;
  transition: all 0.3s ease;
}

.popup-content button.restart {
  background-color: #4CAF50; /* 绿色背景 */
}

.popup-content button.restart:hover {
  background-color: #45a049; /* 深绿色背景 */
}


.normal {
  background-color: #A8A878;
}

.fire {
  background-color: #F08030;
}

.water {
  background-color: #6890F0;
}

.electric {
  background-color: #F8D030;
}

.grass {
  background-color: #78C850;
}

.ice {
  background-color: #98D8D8;
}

.fighting {
  background-color: #C03028;
}

.poison {
  background-color: #A040A0;
}

.ground {
  background-color: #E0C068;
}

.flying {
  background-color: #A890F0;
}

.psychic {
  background-color: #F85888;
}

.bug {
  background-color: #A8B820;
}

.rock {
  background-color: #B8A038;
}

.ghost {
  background-color: #705898;
}

.dragon {
  background-color: #7038F8;
}

.dark {
  background-color: #705848;
}

.steel {
  background-color: #B8B8D0;
}

.fairy {
  background-color: #EE99AC;
}

.border-normal {
  border: 4px solid rgba(175, 175, 129, 0.5);
  border-radius: 10px;
}

.border-fire {
  border: 4px solid rgba(240, 128, 48, 0.5);
  border-radius: 10px;
}

.border-water {
  border: 4px solid rgba(104, 144, 240, 0.5);
  border-radius: 10px;
}

.border-electric {
  border: 4px solid rgba(241, 202, 46, 0.5);
  border-radius: 10px;
}

.border-grass {
  border: 4px solid rgba(120, 200, 80, 0.5);
  border-radius: 10px;
}

.border-ice {
  border: 4px solid rgba(152, 216, 216, 0.5);
  border-radius: 10px;
}

.border-fighting {
  border: 4px solid rgba(192, 48, 40, 0.5);
  border-radius: 10px;
}

.border-poison {
  border: 4px solid rgba(160, 64, 160, 0.5);
  border-radius: 10px;
}

.border-ground {
  border: 4px solid rgba(225, 191, 96, 0.5);
  border-radius: 10px;
}

.border-flying {
  border: 4px solid rgba(168, 144, 240, 0.5);
  border-radius: 10px;
}

.border-psychic {
  border: 4px solid rgba(248, 88, 136, 0.5);
  border-radius: 10px;
}

.border-bug {
  border: 4px solid rgba(168, 184, 32, 0.5);
  border-radius: 10px;
}

.border-rock {
  border: 4px solid rgba(184, 160, 56, 0.5);
  border-radius: 10px;
}

.border-ghost {
  border: 4px solid rgba(112, 88, 152, 0.5);
  border-radius: 10px;
}

.border-dragon {
  border: 4px solid rgba(112, 56, 248, 0.5);
  border-radius: 10px;
}

.border-dark {
  border: 4px solid rgba(112, 88, 72, 0.5);
  border-radius: 10px;
}

.border-steel {
  border: 4px solid rgba(184, 184, 208, 0.5);
  border-radius: 10px;
}

.border-fairy {
  border: 4px solid rgba(238, 153, 172, 0.5);
  border-radius: 10px;
}

.lives img {
  width: 30px;
  height: 30px;
  margin: 0 5px;
}
</style>

