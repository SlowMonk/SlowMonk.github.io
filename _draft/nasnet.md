---
layout: post
title: "CIFAR-10 정복 시리즈 8: NASNet"
subtitle: "Learning Transferable Architectures for Scalable Image Recognition"
categories: 
tags: 
comments: true
---

## CIFAR-10 정복하기 시리즈 소개
CIFAR-10 정복하기 시리즈에서는 딥러닝이 CIFAR-10 데이터셋에서 어떻게 성능을 높여왔는지 그 흐름을 알아본다. 또한 코드를 통해서 동작원리를 자세하게 깨닫고 실습해볼 것이다. 

- CIFAR-10 정복하기 시리즈 목차(클릭해서 바로 이동하기)
  - [CIFAR-10 정복 시리즈 0: 시작하기](https://dnddnjs.github.io/cifar10/2018/10/07/start_cifar10/)
  - [CIFAR-10 정복 시리즈 1: Batch-Norm](https://dnddnjs.github.io/cifar10/2018/10/08/batchnorm/)
  - [CIFAR-10 정복 시리즈 2: ResNet](https://dnddnjs.github.io/cifar10/2018/10/09/resnet/)
  - [CIFAR-10 정복 시리즈 3: DenseNet](https://dnddnjs.github.io/cifar10/2018/10/11/densenet/)
  - [CIFAR-10 정복 시리즈 4: Wide ResNet](https://dnddnjs.github.io/cifar10/2018/10/12/wide_resnet/)
  - [CIFAR-10 정복 시리즈 5: Shake-shake](https://dnddnjs.github.io/cifar10/2018/10/13/shake_shake/)
  - [CIFAR-10 정복 시리즈 6: PyramidNet](https://dnddnjs.github.io/cifar10/2018/10/24/pyramidnet/)
  - [CIFAR-10 정복 시리즈 7: Shake-Drop](https://dnddnjs.github.io/cifar10/2018/10/19/shake_drop/)
  - [CIFAR-10 정복 시리즈 8: NASNet](https://dnddnjs.github.io/cifar10/2018/11/03/nasnet/)
  - [CIFAR-10 정복 시리즈 9: ENAS](https://dnddnjs.github.io/cifar10/2018/11/03/enas/)
  - [CIFAR-10 정복 시리즈 10: Auto-Augment](https://dnddnjs.github.io/cifar10/2018/11/05/autoaugment/)

- 관련 코드 링크
  - [pytorch cifar10 github code](https://github.com/dnddnjs/pytorch-cifar10) 

  
## 논문 제목: Learning Transferable Architectures for Scalable Image Recognition [2017 Jul]

<img src="https://www.dropbox.com/s/hz80yrhndj8h91d/Screenshot%202018-11-03%2015.56.45.png?dl=1">
- 논문 저자: Barret Zoph, Vijay Vasudevan, Jonathon Shlens, Quoc V. Le
- 논문 링크: [https://arxiv.org/pdf/1707.07012.pdf](https://arxiv.org/pdf/1707.07012.pdf)

Auto-Augment를 구현하기 위해서 이 논문을 보게 됐다. 잘 보면 Auto-Augment와 저자가 3명이 겹친다. Auto-Augment 논문에서 어떻게 강화학습을 했는지 거의 정보를 주지 않는다. 하지만 사실 Auto-Augment의 학습 방식와 Neural Architecture Search의 학습방식은 동일할 수 밖에 없다. 따라서 저자들이 기존에 NAS에서 강화학습을 어떻게 사용했는지를 보고 Auto-Augment에 그 방식을 적용하면 될 것 같다. 현재로서는 NAS보다도 Auto-Augment가 CIFAR-10 데이터에서 accuracy가 높다.

<br/>

### Abstract
- image classification 모델을 만드려면 많은 엔지니어링을 해야한다.
- 따라서 우리는 buliding block을 알아서 search하는 방식을 소개한다.
- 그냥 search하면 상당히 오래 걸리기때문에 작은 dataset에 대해서 모델을 학습한 다음에 큰 dataset으로 transfer 하는 방식을 사용한다.
  - transfer 하는 방식을 사용하기위해 NASNet search space을 정의한다.
  - CIFAR-10에서 가장 좋은 convolutional layer를 찾은 다음에 이 layer 혹은 cell을 stack 해서 쌓아서 ImageNet 데이터에 적용한다.
- 새로운 regularization 방법도 사용했다. ScheduledDropPath 라는 방법이며 이 방법이 NASNet model의 일반화 성능을 상당히 개선했다. 
- CIFAR-10 데이터에서 가장 낮은 error는 2.4 % 이다. 
- ImageNet에서 SoTA 달성. 모바일용으로 만든 small NASNet도 다른 같은 사이즈의 네트워크랑 비교했을 때 SoTA 달성.
- NASNet으로부터 학습한 feature를 Faster-RCNN에 사용했을 때도 SoTA

<br/>

### Introduction
- 자동으로 모델 구조를 찾아주는 건 보통 Neural Architecture Search라고 부르며 강화학습을 사용해왔다. 
- 하지만 기존에는 큰 데이터셋에 바로 적용했기 때문에 계산량이 너무 많았다. 
- 따라서 이 논문에서는 가벼운 CIFAR-10 데이터셋에 먼저 학습하고 그 다음에 ImageNet으로 확장하는 방법을 소개하는 것이다. (논문 이름에 왜 scalable, transferable 이라는 단어가 들어가는지 알겠다)
- 우리는 새로운 search space를 정의했다. 이 search space를 정의할 때 network의 깊이나 이미지의 사이즈 때문에 architecture가 더 복잡해지지 않도록 했다. 
- 즉 하나의 convolution block 혹은 cell의 구조를 찾는 것이 목적이다. 구조는 같아도 weight은 다를 수 있는 거니까.
- 우리의 주된 결과는 CIFAR-10에서 학습한 NASNet을 ImageNet에 적용했을 때 SoTA를 달성했다는 것이다. CIFAR-10에서도 SoTA.

<br/>

### Related Work
- 간단히 말해서 이 문제는 hyperparameter optimizationm
- 최근에 나온 방법은 Neural Fabrics, DiffRNN, MetaQNN, DeepArchitect 등이 있다.
- evolutionary algorithm이 좀 더 flexible하다. 하지만 스케일이 클때는 잘 못함.
- 하나의 네트워크가 다른 네트워크와 소통하면서 학습하는 것은 일종의 meta-learning이다. (오호... 무슨 말이지)
- 우리가 이 논문에서 search space를 정의할 때 LSTM과 Neural Architecture Search Cell에서 많은 영감을 얻었다. 

<br/>

### Method
- NAS에 강화학습을 쓴 논문이 있다. "Neural architecture search with reinforcement learning"인데 이 논문의 저자들이다.
  - NAS에서는 controller라는 게 있는데 이건 RNN이다. Controller는. child라고 부르는 네트워크를 여러 다른 architecture로 샘플링한다.
  - 각 child model은 dataset에 대해서 학습하고 validation set에 대해서 평가한다. 
  - child model의 평가결과를 통해 controller를 업데이트한다. 업데이트할. 때는 policy gradient를 사용했다. 아래 그림 참고

  <img src="https://www.dropbox.com/s/xzm0e2k5mn0sj8f/Screenshot%202018-11-03%2016.32.58.png?dl=1">

- 이 논문의 주된 contribution은 데이터셋끼리 transfer 할 수 있도록 search space를 정의했다는데 있다. 이 search space를 우리는 NASNet search space라고 부른다. 
- 우리는 미리 전체 네트워크 구조를 정해놨다. 전체 네트워크는 convolutional cell의 반복으로 되어있다. CIFAR-10과 ImageNet에서 모델 구조는 다음 그림과 같다. cell은 Normal cell과 Reduction cell 두 가지가 있다. Reduction cell은 ResNet의 bottleneck block과 같은 것이다. feature size가 반으로 주는 것을 의미하고 stride=2를 사용한다.

<img src="https://www.dropbox.com/s/7y550jf2oujp97n/Screenshot%202018-11-03%2021.33.13.png?dl=1"> 

- 결국 controller RNN이 하는 것이 이 cell이라는 놈의 구조를 찾는 것이다. 위의 그림을 보면 cell이라는게 정해지면 그냥 그 cell을 쌓는 것이기 때문이다. 어떻게 찾을까? 단순히 보면 다음 그림과 같다. 
- cell 이라는 것은 5개의 block으로 이루어져있다. 다음 그림은 각 block을 만드는 방법이다. 하나의 block을 만들기위해 RNN은 5번의 softmax 출력을 한다.  
<img src="https://www.dropbox.com/s/w0lpjm2004dih33/Screenshot%202018-11-03%2021.38.37.png?dl=1">

- 5번의 softmax 출력은 다음과 같다. 이전 block으로부터 생성된 hidden states 들이란 것은 쉽게 말해 이전 block의 output이다. 이렇게 하는 이유는 어느정도 residual connection의 구조도 가질 수 있도록 하기 위해서이다. 이 block의 구조는 2갈래로 갈라지는 ResNeXt 같은 구조라고 보면 된다. 
  - 1. Select a hidden state from hi, hi−1 or from the set of hidden states created in previous blocks.
  - 2. Select a second hidden state from the same options as in Step 1.
  - 3. Select an operation to apply to the hidden state selected in Step 1.
  - 4. Select an operation to apply to the hidden state selected in Step 2.
  - 5. Select a method to combine the outputs of Step 3 and 4 to create a new hidden state

- 위의 step3나 step4에서는 hidden state에 적용될 operation을 고르는데 다음 중에서 고른다. 
<img src="https://www.dropbox.com/s/wzkyrdjcrz0h2sz/Screenshot%202018-11-03%2021.49.44.png?dl=1">
- 마지막 5스텝에서는 두 갈래의 hidden state를 어떻게 합칠지에 대한 선택이다. element-wise addition을 할 수도 있고 concatenation을 할 수도 있다. 
- 이렇게 만들어진 block의 예시는 다음과 같다. 
<img src="https://www.dropbox.com/s/umibxvdkjjd7oxe/Screenshot%202018-11-03%2021.51.05.png?dl=1">

- 각 cell은 5개의 block으로 이루어져있다. RNN은 Normal Cell과 Reduction Cell에 대해서 둘 다 prediction을 한다. 따라서 RNN은 2x5x5 개의 prediction을 한다고 볼 수 있다.

<br/>

### Experiments and Results
- controller RNN은 PPO를 사용해서 학습한다. 
- 전체 학습은 500 GPU를 사용해서 4일이 걸렸다.
- 최종 학습된 best cell은 세가지가 있는데 그중 NASNet-A는 다음과 같다. 
<img src="https://www.dropbox.com/s/87iqnvi9j0j3o3c/Screenshot%202018-11-03%2022.19.06.png?dl=1">
- cell의 구조가 정해지더라도 정해야할게 몇가지 더 있다. cell을 몇 번씩 반복할지에 대한 N, 첫 conv의 filter 수이다.
- 학습 방법은 ScheduledDropPath를 사용. effective regularization 방법으로 사용됌. 간단하게 말하면 cell 안에 있는 path 중에 일부를 dropout처럼 drop 할건데 학습과정중에서 뒤로 갈수록 drop rate를 떨어뜨리는 것이다. 
- CIFAR-10에서는 N=4 or 6로 사용. error rate 2.4%로 SoTA. 결과는 다음 표를 참고
<img src="https://www.dropbox.com/s/vnql39bxuhr2vpd/Screenshot%202018-11-03%2022.27.29.png?dl=1">

<br/>

### Experimental Details
- controller architecture
  - controller RNN은 1-layer lstm이다. hidden unit은 100이다. output은 10B개의 prediction이 출력된다. 한 episode의 길이는 그렇다면 10B인걸까? 
  - 10B의 softmax의 모든 확률을 쭉 곱하면 child model을 선택할 확률이 나온다. 이 확률은 gradient 계산할 때 사용된다. gradient는 validation accuracy에 의해 scale된다. 
  - PPO의 learning rate는 0.00035, entropy penalty weight는 0.00001, reward는 0.95의 weight로 moving average 된다.
- training of the controller
  - controller RNN을 update 할때는 20개의 architecture의 결과를 모아서 했다. 
  - 학습은 20,000개의 architecture를 테스트하는 것으로 진행된다. 학습이 끝나면 250개의 top architecture를 뽑아서 수렴할 때까지 CIFAR-10에서 돌린다.
- details of architecture search space
  - activation function은 relu로 통일
  - 다음 그림과 같이 block이 올라갈수록 hidden state set은 늘어난다. 
  <img src="https://www.dropbox.com/s/ssxcf5vys5n8844/Screenshot%202018-11-03%2022.49.50.png?dl=1">
- CIFAR-10에 학습할 때는 optimizerd의 momentum=0.9로 사용
- 모든 모델은 L2 weight decay 사용
- 20 epoch만 학습. 20 epoch 동안 cosine learning rate decay
- search 하는 동안에는 N=2로 사용. 그렇게 해도 cell 구조 찾는데는 문제없음.