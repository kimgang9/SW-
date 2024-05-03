<h1>DiffusionAD(99.700)</h1>
<p>
오토 인코더 기반 네트워크(AE)를 사용하며, 인코더가 입력 이미지를 저차원 표현으로 압축한 후 디코더가 비정상 영역을 정상으로 재구성한다는 가정을 기반으로 한다.
</p>
<p>
문제점<br>
1. 원본 이미지에서 압축된 저차원 표현에는 여전히 비정상적인 정보가 포함되어 있기 때문에 비정상 영역을 불변으로 재구성할 수 있으며, 잘못된 음성 감지가 발생할 수 있다. <br>
2. AE(autoencoderbased networks)는 제한된 복원 기능으로 인해 정상 영역을 거칠게 재구성할 수 있으며, 특히 복잡한 구조 또는 텍스처가 있는 데이터 셋에서 많은 잘못된 양성을 도입할 수 있다. <br><br>

해결책<br>
- 입력 이미지를 교란시키기 위해 가우시안 노이즈 도입<br>
- 추가된 노이즈를 예측하기 위해 노이즈 제거 모델을 사용하여 재구성 프로세스를 노이즈 투 노름 패러다임(noise-to-norm paradigm)으로 재구성<br>
</p>
<p>
무작위로 샘플링된 가우시안 노이즈에서 실제 AD(Anomaly Detection) 시나리오의 실시간 요구 사항보다 훨씬 느리다.<br>
해결책 : 확산 모델을 사용하여 노이즈를 한 번 예측한 다음 재구성 결과를 직접 예측하는 이상 감지를 위한 1단계 노이즈 제거 패러다임을 사용한다. <br>
이렇게 하면 다양한 데이터 셋과 이상 유형의 이상에 걸쳐 원하는 이상 없는 재구성을 달성하는 것을 관찰할 수 있다. 그 후 분할 네트워크는 정확하게 예측 입력과 재구성 간의 불일치 및 공통점을 활용하여 픽셀 수준 이상 점수를 산출한다.
</p>

<h3>요약</h3>
- 노이즈 투 노름 패러다임을 통해 입력 이미지를 이상 없는 복원으로 재구성하고, 이들 간의 불일치와 공통점을 활용하여 픽셀 단위의 이상 점수를 추가로 예측하는 새로운 파이프라인인 확산 AD를 제안함<br>

<h3>데이터 셋(4가지)</h3>
1. MVTec(4096개의 정상 이미지와 1258개의 비정상 이미지)<br>
- 스크래치, 균열, 구멍 및 함몰부와 같은 다양한 유형의 표면 결함 샘플이 포함된다.<br>
2. VisA(9621개의 정상 이미지와 1200개의 비정상 이미지)<br>
- 긁힘, 움푹 들어간 곳, 착색된 반점 또는 균열과 같은 표면 결함과 부품의 잘 못 배치 또는 누락과 같은 구조적 결함을 포함하여 다양한 불완전성이 포함된다.<br>
3. DAGM(15000개의 정상 이미지와 2100개의 비정상 이미지)<br>
- 스크래치, 얼룩과 같이 시각적으로 배경에 가까운 다양한 결함이 비정상 샘플을 구성한다.<br>
4. MPDD(1064개의 정상 이미지와 282개의 비정상 영상)<br>
- 금속 가공에 초점을 맞추고 수동으로 작동하는 생산라인에서 마주치는 실제 상황을 반영한다.<br>


<h3>결론</h3>
비정상적인 영역이 가우시안 노이즈에 의해 교란된 후 다음으로 재구성되는 노이즈 대 노름 패러다임을 채택하는 확산 모델을 사용한 재구성 프로세스이다.<br>
확산 모델의 반복적인 노이즈 제거 접근 방식보다 훨씬 빠른 1단계 노이즈 제거 패러다임을 제시하며, 재구성 품질을 더욱 향상시키기 위해 규범 유도 패러다임을 제안한다.<br>
마지막으로, 분할 하위 네트워크는 불일치와 공통점을 활용한다.<br><br>

```Python
# 주어진 확률 분포로부터 이미지를 샘플링하고, 가우시안 노이즈를 추가하여 샘플링된 이미지를 생성하는 과정
def sample_p(self, model, x_t, t, denoise_fn="gauss"): 
    out = self.p_mean_variance(model, x_t, t) # 입력 이미지 x_t와 시간 변수 t에 대한 확률 분포의 평균과 분산 계산
    if denoise_fn == "gauss": # 노이즈 함수가 가우시안인 경우, 이미지와 같은 크기의 가우시안 노이즈를 생성
        noise = torch.randn_like(x_t) 
    else: # 노이즈 함수가 가우시안이 아니면, 사용자가 지정한 노이즈 함수 denoise_fn을 사용하여 노이즈 생성
        noise = denoise_fn(x_t, t)
    # 시간 변수가 0이 아닌 위치를 나타내는 마스크를 생성
    nonzero_mask = ( (t != 0).float().view(-1, *([1] * (len(x_t.shape) - 1))))
    # 확률 분포의 평균과 분산을 이용하여 가우시안 노이즈를 추가한 후 이미지의 확률적 샘플을 생성
    sample = out["mean"] + nonzero_mask * torch.exp(0.5 * out["log_variance"]) * noise 
    # 샘플된 이미지와 초기 이미지의 예측값을 반환
    return {"sample": sample, "pred_x_0": out["pred_x_0"]}
```
<br><br>
```Python
# 데이터의 노이즈를 제거하여 초기 예측을 개선하고, 이상 감지를 위한 더 강력한 시스템을 구축하기 위한 단계
def norm_guided_one_step_denoising(self, model, x_0, anomaly_label,args):
        # 주어진 범위 내에서 무작위로 낮은 스케일의 시간 단계를 생성
        normal_t = torch.randint(0, args["less_t_range"], (x_0.shape[0],),device=x_0.device)
	# 주어진 범위 내에서 무작위로 높은 스케일의 시간 단계를 생성
        noisier_t = torch.randint(args["less_t_range"],self.num_timesteps,(x_0.shape[0],),device=x_0.device)
        
	# 낮은 스케일의 시간 단계에 대한 손실, 초기 예측, 예상되는 노이즈를 계산
        normal_loss, x_normal_t, estimate_noise_normal = self.calc_loss(model, x_0, normal_t)
	# 높은 스케일의 시간 단계에 대한 손실, 초기 예측, 예상되는 노이즈를 계산
        noisier_loss, x_noiser_t, estimate_noise_noisier = self.calc_loss(model, x_0, noisier_t)
        
	# 높은 스케일의 시간 단계에서 예측된 초기 이미지를 얻음
        pred_x_0_noisier = self.predict_x_0_from_eps(x_noiser_t, noisier_t, estimate_noise_noisier).clamp(-1, 1)
	# 낮은 스케일의 시간 단계에서 예측된 초기 이미지를 샘플링함
        pred_x_t_noisier = self.sample_q(pred_x_0_noisier, normal_t, estimate_noise_normal)   

        # 정상 레이블에 대해서만 손실을 계산하고 평균을 취함
        loss = (normal_loss["loss"]+noisier_loss["loss"])[anomaly_label==0].mean()
	# 손실이 NaN인 경우 손실을 0으로 채움
        if torch.isnan(loss):
            loss.fill_(0.0)

	# 예상되는 노이즈의 수정된 버전 계산
        estimate_noise_hat = estimate_noise_normal - extract(self.sqrt_one_minus_alphas_cumprod, normal_t, x_normal_t.shape, x_0.device) * args["condition_w"] * (pred_x_t_noisier-x_normal_t)
	# 수정된 예상 노이즈를 사용하여 초기 이미지에서 노이즈를 제거한 최종 초기 예측을 계산
        pred_x_0_norm_guided = self.predict_x_0_from_eps(x_normal_t, normal_t, estimate_noise_hat).clamp(-1, 1)

        return loss,pred_x_0_norm_guided,normal_t,x_normal_t,x_noiser_t
```
