# Интерпретация работы REINVENT4

Модель:

$$P_\theta(T \mid S) = \prod_{i=1}^{L} P_\theta(t_i \mid t_{\text{<}i}, S)$$

Во время обучения минимизируется кросс-энтропийная функция потерь:

$$\mathcal{Loss} = -\sum_{(S,T)\in D}\ln P_\theta(T \mid S)$$

где $D$ — набор пар «исходник–аналог» с известным уровнем сходства.

## Сэмплирование `Mol2Mol`

Mol2Mol — это условная генеративная модель, которая на вход получает исходную молекулу \(S\) и генерирует структурно похожую молекулу \(T\).  
Для генерации используется либо стохастический multinomial-sampling (наш случай), либо beam-search. При этом распределение токенов корректируется температурой $T$:

$$P_T(t_i) = \frac{\exp\bigl(\ln P_\theta(t_i \mid t_{\text{<}i}, S)/T\bigr)} {\sum_{u}\exp\bigl(\ln P_\theta(u \mid t_{\text{<}i}, S)/T\bigr)}$$

- $T<1$ делает распределение более «острым» и консервативным.  
- $T>1$ сглаживает распределение, увеличивая разнообразие.

В beam-search храним $k$ лучших кандидатов по сумме лог-скор:

$$\text{score}(t_{1:i}) = \sum_{j=1}^{i} \ln P_\theta\bigl(t_j \mid t_{\text{<}j}, S\bigr).$$

### Pipeline использования

1. Вход: список SMILES референсных лигандов $S$.  
2. Генерация: из каждого $S$ получаем аналог $T\sim P_T(\cdot\mid S)$.  
3. Выход: набор молекул $\{T\}$, готовых к downstream-скрингу и оптимизации.

Prior `mol2mol_medium_similarity.prior` был предварительно обучен на парах $(S,T)$ с контролем Tanimoto-сходства, поэтому модель умеет вносить химически обоснованные изменения (R-группы, кольца, хиральные центры) без потери заданного профиля похожести.

---

## Параметры scoring`а

$$\text{Reward} = \left( \prod_{i=1}^{N} S_i^{w_i} \right)^{\frac{1}{\sum_{i=1}^{N} w_i}}$$

где:  
- $S_i$ — скор компонента после трансформации  
- $w_i$ — вес компонента  
- $N$ — число активных компонентов (вес $> 0$ )  

**OR**

$$\text{Reward} = \left(\prod_{i=1}^{N} \bigl(T_{i}(x_{i})\bigr)^{w_{i}}\right)^{\frac{1}{\sum_{i=1}^{N}w_{i}}} \ =\  \exp\ \Biggl(\frac{\sum_{i=1}^{N}w_{i}\,\ln\bigl(T_{i}(x_{i})\bigr)}{\sum_{i=1}^{N}w_{i}}\Biggr)$$

где:  
- $x_{i}$ — сырое значение $i$-го дескриптора,  
- $T_{i}(x_{i})$ — трансформация ($\text{sigmoid}$, $\text{step}$, $\text{value mapping}$ и т. д.) нормирующая $x_{i}$ в $(0,1]$,  
- $w_i$ — вес компонента,  
- $N$ — число активных компонентов (вес $> 0$ )  

[CONFIG FILE](./configs/REINVENT4_scoring_config.toml)

| 💡 Категория                | 🧩 Компонент                         | ⚖️ Вес wᵢ | 🔄 Трансформация                                                                 | 💬 Комментарий                                        | 📏 Химико-биологическая интерпретация диапазонов                |
|------------------------------|-------------------------------------|-----------|----------------------------------------------------------------------------------|-------------------------------------------------------|----------------------------------------------------------------|
| Drug-likeness                | QED                                 | 1.0       | —                                                                                | Общая «медикаментозность»                             |        |
| Lipophilicity                | SlogP (RDKit)                       | 1.0       | reverse_sigmoid (low = 1.4, high = 3.0, k = 0.4)                                  | Липофильность                                          | logP 1.4–3.0 — оптимум между растворимостью и проницаемостью  |
| Size & Permeability          | MolecularWeight                     | 1.0       | double_sigmoid (low = 200, high = 500, coef_div = 500, coef_si = 20, coef_se = 20) | Молекулярная масса                                     | 200–500 Da — оптимум для пероральной биодоступности и BBB       |
| Polarity & BBB               | TPSA                                | 0.9       | double_sigmoid (low = 20, high = 90, coef_div = 70, coef_si = 15, coef_se = 15)   | Полярная поверхность                                   | < 90 Å² — высокая BBB-проницаемость; < 140 Å² — пероральная доступность    |
| Stereochemistry              | Number of stereo centers            | 0.7       | step (low = 0, high = 3)                                                         | Хиральность                                            | 0–3 центра — контролируемая стереоразнообразность             |
| Hydrogen bonding             | HBondAcceptors                      | 0.8       | reverse_sigmoid (low = 5, high = 8, k = 0.5)                                      | Количество акцепторов H-связей                        | 5–8 — оптимум для селективности и мембранного транспорта      |
| Hydrogen bonding             | HBondDonors                         | 0.8       | reverse_sigmoid (low = 0, high = 3, k = 0.5)                                      | Количество доноров H-связей                            | 0–3 — баланс водородных связей и проницаемости                |
| Flexibility                  | Number of rotatable bonds           | 0.7       | reverse_sigmoid (low = 0, high = 10, k = 0.5)                                     | Гибкость молекулы                                       | ≤ 10 — достаточная мобильность без потери селективности        |
| 3D насыщенность              | Csp3                                | 0.6       | reverse_sigmoid (low = 0, high = 15, k = 0.5)                                     | Доля sp³-углеродов                                     | 0–15 — увеличение объёмности и метаболической стабильности     |
| Hybridization                | Number of sp atoms                  | 0.6       | reverse_sigmoid (low = 0, high = 20, k = 0.5)                                     | Жёсткость фрагментов                                   | sp атома влияют на плоскостность и π-системы                   |
| Hybridization                | Number of sp² atoms                 | 0.4       | reverse_sigmoid (low = 0, high = 20, k = 0.5)                                     | Плоскостность                                          | sp² — формируют π-связи и ароматические системы               |
| Hybridization                | Number of sp³ atoms                 | 0.6       | reverse_sigmoid (low = 0, high = 20, k = 0.5)                                     | 3D-насыщенность                                        | sp³ — увеличивает объём и гибкость                            |
| Size                         | Number of heavy atoms               | 0.6       | reverse_sigmoid (low = 20, high = 64, k = 0.5)                                    | Общий размер молекулы                                   | 20–64 атома — оптимум для drug-likeness и BBB                 |
| Polarity                     | Number of heteroatoms               | 0.6       | reverse_sigmoid (low = 6, high = 30, k = 0.5)                                     | Полярные атомы                                         | 6–30 — баланс полярности и липофильности                      |
| Rigidity                      | Number of rings                     | 0.8       | reverse_sigmoid (low = 0, high = 7, k = 0.5)                                      | Цикличность                                            | 0–7 колец — оптимальная жёсткость и селективность             |
| Aromaticity                  | Number of aromatic rings            | 0.8       | reverse_sigmoid (low = 0, high = 3, k = 0.5)                                      | Ароматические кольца                                   | 0–3 колца — π-π взаимодействия и селективность                |
| Aliphatic character           | Number of aliphatic rings           | 1.0       | reverse_sigmoid (low = 0, high = 3, k = 0.5)                                      | Алифатические циклы                                    | 0–3 колца — влияют на растворимость и стабильность            |
| Molecular volume             | Moleculer Volume (RDKit)            | 1.0       | reverse_sigmoid (low = 300, high = 600, k = 0.5)                                   | Объём молекулы                                         | 300–600 Å³ — баланс между размером и проницаемостью BBB       |
| Filters                      | Group (substructure) count [F,Cl]   | 1.0       | reverse_sigmoid (low = 1, high = 3, k = 0.5)                                      | Счётчик галогенов                                      | > 3 — повышенный риск токсичности и плохой ADME                |
| Filters                      | custom alerts                       | 1.0       | глобальный фильтр                                                                 | SMARTS-паттерны отбора                                | Отсекает токсичные/нестабильные фрагменты                      |
| Similarity                   | Tanimoto similarity to reference    | 0.8       | —                                                                                | ECFP6-сходство                                          | 0.7–1.0 — близкие аналоги с сохранением активности            |
| Synthetic accessibility      | SA score (SAScore)                  | 1.0       | —                                                                                | Синтетическая доступность                              |        |
