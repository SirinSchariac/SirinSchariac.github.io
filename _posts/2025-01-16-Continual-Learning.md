---
layout: 	post  				   
title: 		Continual Learning				
subtitle:	An Intro to continual learning
date:       2025-01-16 				
author:     Sirin 						
header-img: img/universe.jpg
catalog: 	true 				
tags:						
    - Introduction
    - ML
---

### Continual Learning

> #### What is continual learning

一般来说，ML是在一个固定的数据集上针对特定的下流任务来训练一个相应的模型。但对于现实世界的任务而言，环境是在不断变化着的，人的智能是在不断学习的，而之前学过的知识也会保留。因此，Continual Learning被提出，旨在让模型能够从新任务中学习，完成新的目标的同时，保持其在原有数据集上的性能表现。

> #### Objectives of continual learning

- 减少对于旧任务的访问
- 减少模型对于存储和计算需求的增长
- 快速适应新任务
- 减少灾难性遗忘([Catastrophic Forgetting](https://en.wikipedia.org/wiki/Catastrophic_interference), CF)，模型在新任务上训练后，不应在旧的任务上性能表现变差
- 保持可塑性，使得模型可以随时学习新的任务

> #### Three Approaches

- Regularization techniques：通过正则化项来保护在之前的任务中学习到的重要参数，e.g., Elastic Weight Consolidation
- Replay methods: 周期性地使用旧任务数据的一部分配合新任务来训练模型
- Parameter isolation methods: 动态地划分网络的“部分”给不同的任务。

> #### Three Scenarios

- **Task-incremental learning**
  此类情况下，任务之间的区别是可以明确获取到的，因此，可以针对每个任务训练单独的输出层甚至单独的模型(后者能完全消除CF问题)，因此，此类场景下的挑战主要是如何高效获取任务之间的shared learned representation
- **Domain-incremental learning**
  此类情况下，需要解决的问题的结构始终是相同的，但问题的内容或者输入分布会发生变化。例如，不同天气、不同光照条件下的目标检测
- **Class-incremental learning**
  在这种情况下，agent需要辨识的类别是随着新任务的到来而逐渐增多的，比如第一个任务需要分辨猫和狗，而第二个任务则需要分辨鼠和兔，这种情况下，agent要学习到没有同时出现的类之间的差异。



#### Reference

[1] M. De Lange et al., "A Continual Learning Survey: Defying Forgetting in Classification Tasks," in IEEE Transactions on Pattern Analysis and Machine Intelligence, vol. 44, no. 7, pp. 3366-3385, 1 July 2022, doi: 10.1109/TPAMI.2021.3057446

[2] van de Ven, G.M., Tuytelaars, T. & Tolias, A.S. Three types of incremental learning. *Nat Mach Intell* **4**, 1185–1197 (2022). https://doi.org/10.1038/s42256-022-00568-3

