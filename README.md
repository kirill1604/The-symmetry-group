# DataCon2025 | Drug Discovery: Generation of HDAC6 inhibitors for the treatment of Alzheimer's disease

## Project structure

- [Final presentation](./docs/Презентация_Datacon_2025_Группа_симметрии.pptx)
- [ITMO lectures and presentations](./docs/lectures/)
- [Useful scripts from ITMO](./exps/oth_ipynb/)
- [Tasks description](./docs/tasks.md)
- [Mini-tasks](./exps/mini_tasks/)
- [Data](./data/)
- [REINVENT4 configurations](./configs/)
- [Dependencies](requirements.md)
- [Documentation templates for preprints and articles](./TechDocs/)
- [Scripts for working with documentation](./tools/)

## Project description

This project is dedicated to generating new molecules that are potentially capable of interacting with the HDAC6 target, which is involved in the pathogenesis of Alzheimer's disease.  
The project was developed as part of the DataCon2025 competition and uses modern machine learning methods to work with chemical data.

Alzheimer's disease (AD) is one of the most common neurodegenerative diseases, characterized by the formation and deposition of amyloid β (Aβ) plaques in the brain. These plaques trigger a cascade of pathological processes: tau protein hyperphosphorylation → neuroinflammation → neuronal degeneration and cognitive decline.  
Although the amyloid cascade remains one of the leading hypotheses for the pathogenesis of the disease, attempts to develop effective drugs against Aβ have not yet been crowned with significant success, mainly due to the biological complexity of the disease. Current research offers new approaches — from biomarkers and genetic strategies to therapeutic diagnostics — but the need for alternative targets remains extremely relevant.

---

## Steps

### 1. Target selection

По итогам [сравнительного анализа сигнальных путей](./docs/targets_comparative_analysis.md) и количества доступных данных в открытых базах данных была выбрана целевая мишень:

> **HDAC6**

Гистондеацетилазы (`HDACs`) — это эпигенетические регуляторы, удаляющие ацетильные группы с гистонов и тубулинов. Среди 18 известных изоформ `HDAC` особое внимание привлекает `HDAC6`.  
Исследования показали, что у пациентов с БА наблюдается повышенная экспрессия `HDAC6`, что в свою очередь связано со снижением уровня ацетилированного α-тубулина и нарушением работы нейронов. Более того, `HDAC6` взаимодействуя с тау-белком, способствует его гиперфосфорилированию и формированию нейрофибриллярных клубков (NFTs).

Интерес к ингибиторам `HDAC6` (`HDAC6i`) усилился после того, как в доклинических моделях БА селективные `HDAC6i` продемонстрировали:

- Снижение уровня тау-белка;
- Восстановление аксонального транспорта;
- Улучшение обучаемости и памяти;
- Противовоспалительное действие.

---

### 2. Datasets and data processing

#### Data

Использовалась база данных `BindingDB`, содержащая **10098** молекул, отфильтрованных по `IC50`.

#### Data processing

1) Предварительная подготовка данных:
   - обработка пропущенных значений и дубликатов;
   - проверка корректности молекул по SMILES;
   - фильтрация по сайту связывания мишени;
   - фильтрация по PDB структурам мишени.

2) Расчет и отбор дескрипторов производился с использованием библиотек `RDKit` и `Padelpy`:
   - рассчитаны молекулярные отпечатки `MACCS Keys` и `Morgan Fingerprints`
   - осуществлена фильтрация и отбор признаков:
     - с некорректными значениями;
     - нулевой дисперсией;
     - высокой корреляцией ($r > 0.7$ между признаками).

---

### 3. Training models to predict biological activity

Использовались модели `XGBoost` и `ChemProp`.

Для [прогнозирования активности молекул](./exps/Prediction.ipynb) с использованием Fingerprints была обучена модель `XGBoostRegressor`. После оптимизации гиперпараметров с помощью `GridSearchCV` были получены следующие метрики на тестовом наборе данных:

- **$R^2$ (коэффициент детерминации):** $0.657$
- **$\text{MAE}$ (средняя абсолютная ошибка):** $0.423$
- **$\text{RMSE}$ (среднеквадратичная ошибка):** $0.582$

---

### 4. Molecules generation

Для генерации молекул использовалось несколько моделей:

#### [IDOLpro](https://github.com/sandbox-quantum/idolpro)

Основана на условной диффузионной модели `DiffSBDD-Cond`, учитывающей симметрии SE(3) и обученной генерации лигандов с высокой аффинностью к заданному белковому карману. За 2 ч 40 мин сгенерировать новые лиганды не удалось, поэтому было решено воспользоваться другой моделью.

#### [DRAGONFLY](https://github.com/ETHmodlab/dragonfly_gen)

Глубокая генеративная модель для создания биоактивных молекул по эталонному лиганду или структуре мишени. Использует анализ drug-target interactome, Graph Transformer Neural Network (GTNN) для кодирования и языковую модель на базе RNN-LSTM для генерации SMILES.

Для генерации использовался файл мишени [8qa7.pdb](./data/8qa7.pdb), так как данная структура имеет высоко разрешение $1.47 \text{Å}$. В качестве лиганда использовался [Trichostatin A](./data/trichostatinA.sdf) благодаря высокой аффинности. $\text{IC₅₀} ≈ 10–11 \text{нМ}$ для человеческого и рыбьего `HDAC6`.

В ходе генерация при помощи `DragonFly` в модель одновременно с исходной молекулой подаются значения шести важных молекулярных свойств. Модель генерирует новые SMILES-структуры так, чтобы эти свойства в новых молекулах почти точно совпадали с заданными.

1) Молекулярная масса (Molecular weight)
2) Число вращающихся связей (Rotatable bonds)
3) Число акцепторов водородных связей (H-bond acceptors)
4) Число доноров водородных связей (H-bond donors)
5) Полярная поверхность (Polar surface area, PSA)
6) Липофильность (MolLogP)

Было сгенерировано **121000** новых [SMILES молекул](./data/sampling_out/DRAGONFLY/)

### [REINVENT4](https://github.com/MolecularAI/REINVENT4)

`REINVENT4` — это командная Python-утилита для генерации малых молекул с помощью алгоритмов глубокого обучения и оптимизации через [многокомпонентное вознаграждение](./docs/REINVENT4_interpretations.md).  

- Генераторы представлены в виде **sequence-based** моделей, оперирующих SMILES-строками;
- Базируются на **рекуррентных нейронных сетях (RNN)** и **трансформерах**, обученных моделировать распределение реальных молекулярных SMILES;
- Алгоритм Reinforcement Learning (RL) использует функцию вознаграждения, заданную пользователем как геометрическое среднее нескольких дескрипторов;
- Поддерживается **Transfer Learning (TL)** и **Curriculum Learning (CL)** для тонкой настройки и поэтапного обучения.

**Ключевые возможности:**

- de novo дизайн молекул «с нуля»;
- **Scaffold hopping** — поиск новых каркасов;
- **R-group replacement** — замена заместителей в известных лигандах;
- **Linker design** — проектирование соединителей между фрагментами;
- **Molecule optimization** — улучшение свойств целевых соединений;
- **Плагины** для кастомных scoring-компонентов;
- Поддержка нескольких стратегий генерации молекул:
  - **de novo sampling** — генерация молекул с нуля, без опоры на исходные фрагменты;
  - **LibInvent** — подбор R-групп для заданных скелетов с точками присоединения;
  - **LinkInvent** — создание линкеров, соединяющих два заданных фрагмента;
  - **Mol2Mol** — генерация аналогов, структурно похожих на входные молекулы;
  - **PepInvent** — дизайн и оптимизация коротких пептидов.

Каждый режим требует соответствующего `.prior` файла и входного набора SMILES (при необходимости). Поддерживаются стохастическая стратегия `multinomial` (с параметром `temperature`) и детерминированная `beamsearch`. Выход сохраняется в CSV с оценкой вероятности (NLL), может быть дополнительно обработан и отсортирован с помощью `scoring`.  

Для [генерации](./configs/REINVENT4_sampling_config.toml) был выбран режим `Mol2Mol`, для которого были отобраны [топ 57 ингибиторов](./data/HDAC6_filtered.csv) `HDAC6`, отсортированных по `pIC50` из общего [очищенного датасета](./data/HDAC6_cleaned.csv)

Итоги генерации в папке [data/sampling_out/REINVENT4/](./data/sampling_out/REINVENT4/)  
Итоги скоринга для [одной из итераций](./data/sampling_out/REINVENT4/sampling_out1.csv) — [data/HDAC6_score.csv](./data/HDAC6_score.csv)  <!-- TODO: Тут нет pIC50 предикта, сматчите колонку из др. файла -->

```python
filtered = cleaned[cleaned["pIC50"] >= 8.7].sort_values(by="pIC50", ascending=False, ignore_index=True)
```

---

### 5. Molecules filtration

<!-- TODO: ДОПОЛНИТЬ  -->

Сгенерированные молекулы проходили [проверку](./exps/Filter.ipynb) по 6 параметрам:

1) QED (Quantitative Estimate of Drug-likeness), критерий: $>= 0.5$;
2) SA score (Synthetic Accessibility Score), критерий: $2 < ... < 6$;
3) Toxicophore, критерий: $0$ токсичных фрагментов;
4) pValue (-log(IC50)), критерий: $> 6$;
5) число нарушений 5 правил Липински $<= 1$;
6) BBB (Blood–Brain Barrier), возможно два варианта '+'(проницаема)/'-'(не проницаема).

Молекула проходила отбор успешно, если хотя бы 5 из 6 параметров попадали в допустимые области. Таких молекул было 20 штук.
Дополнительно было принято решение отсеить все молекулы с ненулевым Toxicophore. Итого: 10 молекул, из них 1 удовлетворяет всем шести критериям, 9 — пяти.

---

### TODO: Possible improvements

В дальнейшем предполагается произвести расчеты докинга и молекулярной динамики при появлении требуемых вычислительных ресурсов для понимания карманов связывания, вычислить полный профиль аффинности для всех возможных мишеней, а также оптимизировать параметры будущих соединений-кандидатов для минимизации нецелевого связывания и тонкой подстройки смещенного ингибирования/(обратной)активации (biased inhibition/(reverse)agonism) конкретных молекулярных цепей, следующих за HDAC6 → ...
Возможно проведение расчетов взаимодействий с G-белками, добавление QSAR-предиктивной модели в reward + дообучение REINVENT4, добавление штрафов за токсичность по дополнительным метрикам (hERG и прч.), а также интеграция моделей для планирования шагов синтеза в пайплайн, наподобие [aizynthfinder](https://github.com/MolecularAI/aizynthfinder).

---

![contributors](./img/contributors.png)
