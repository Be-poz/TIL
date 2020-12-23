# 대표적인 Sort 알고리즘과 그 특징에 대해

<br/>

## 선택 정렬 (Selection Sort)

1. 순차 탐색을 해서 최소/최대 값을 찾는다. 
2. 해당 값을 현재 인덱스의 값과 바꿔준다.
3. 다음 인덱스에서 위의 과정을 반복해준다.

![1](https://user-images.githubusercontent.com/45073750/102910289-aaccfa80-44bd-11eb-91f9-ada370c3f68d.png)

첫 회차 때에 n-1개로 시작해서 1개씩 그 수가 줄어들며 n-2개, ... , 1개를 탐색하게 된다.  
시간 복잡도는 배열과 상관없이 전체 비교를 하여 항상 **O(N^2)** 가 된다.  

```java
    public static void selectionSort(int[] arr) {
        int index;

        for (int i = 0; i < arr.length - 1; i++) {
            index = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[j] < arr[index]) {
                    index = j;
                }
            }
            int temp = arr[i];
            arr[i] = arr[index];
            arr[index] = temp;
        }
    }
```

<br/>

<br/>

## 삽입 정렬 (Insertion Sort)

1. 시작 시, 가장 왼쪽의 수는 정렬이 된 것으로 여기고 그 다음 수 부터 비교를 시작한다.
2. 해당 인덱스의 수를 뽑아 왼쪽에 존재하는 정렬된 배열들과 비교를 한다.
3. 해당 배열의 값 보다 크면 해당 배열 인덱스에 뒤에 삽입한다.
4. 해당 배열의 값 보다 작으면 다음 배열의 값과 비교한다.

![1](https://user-images.githubusercontent.com/45073750/102903504-c501db00-44b3-11eb-8696-6ede7e05cc0b.png)

어느정도 정렬이 되어있다면 그만큼 swap을 할 필요가 없기 때문에 최선의 경우 **O(N)**의 시간복잡도를 가진다.  
그러나 최악의 경우 **O(N^2)**를 가질 수도 있다. 데이터의 상태에 따라 성능의 편차가 심하다.  

```java
    public static void insertionSort(List<Integer> list) {
        int size = list.size();

        for (int i = 1; i < size; i++) {
            for (int j = i - 1; j >= 0; j--) {
                if (list.get(i) > list.get(j)) {
                    list.add(j + 1, list.remove(i));
                    break;
                }
                if (j == 0) {
                    list.add(0, list.remove(i));
                }
            }
        }
    }
```

<br/>

<br/>

## 버블 정렬 (Bubble Sort)

오름차순 기준으로 설명하겠다.

1. 한 쪽 끝에서부터 인덱스를 잡고 시작한다.
2. 해당 인덱스와 그 다음 인덱스를 비교하면서 앞의 값이 크면 바꿔준다.
3. 그 다음 인덱스로 이동한다.
4. 1 ~ 3번을 반복하는데 가장 우측은 정렬이 되어있으니 n - (회전한 수) 만큼 반복한다.

![1](https://user-images.githubusercontent.com/45073750/102912615-f7fe9b80-44c0-11eb-94df-c772b3ddd4b9.png)

버블정렬 또한 기존의 배열이 얼마나 정렬되어있는지 상관없이 전체 비교를 진행하므로 시간복잡도는 **O(N^2)** 이다.  

<br/>

<br/>

 ## 퀵 정렬 (Quick Sort)

1. pivot으로 배열의 값 하나를 정한다. 맨 앞이나 뒤 또는 배열의 중간 값을 보통 둔다.
2. 분할 전, 배열의 왼쪽 끝, 오른쪽 끝의 인덱스를 저장하는 변수를 생성한다.
3. 오른쪽 인덱스부터 비교를 시작하고 그 위치가 왼쪽 인덱스보다 클 때에만 반복한다. 비교 값이 pivot보다 크면 right을 하나 감소하고 비교를 반복한다. pivot 보다 작은 것을 찾으면 반복을 중지한다.
4. left<right 일 때에만 반복하며, 배열 값이 pivot 보다 작으면 left 를 증가시키고 비교를 반복한다. 큰 값을 찾으면 반복을 중지한다.
5. left 인덱스 값과 right 인덱스 값을 바꿔준다.
6. 위 과정을 left < right를 만족할 때 까지 반복한다.
7. 위 과정이 끝나면 left의 값과 pivot을 바꿔준다.
8. 맨 왼쪽에서부터 left - 1 까지, left + 1 부터 맨 오른쪽까지로 나눠 퀵 정렬을 반복한다.

![1](https://user-images.githubusercontent.com/45073750/103007914-9816fc00-4577-11eb-8c29-3ac2434175bd.png)

```java
public class QucickTest {
	public static void main(String[] args) {
		int[] array = { 7, 6, 8, 1, 2, 3, 5, 5, 4 };
		quickSort(array, 0, array.length - 1);
		System.out.println("result = " + Arrays.toString(array));
	}

	public static void quickSort(int[] arr, int start, int end) {
		if (start >= end) { // 검사할 범위가 1일땐 정렬 완료
			return;
		}

		int pivot = start;
		int temp, i, j;
		do {
			i = start + 1; // i++ (pivot 제외 = +1)
			j = end; // j--
			
			// --> 방향으로 피벗값보다 큰값 찾을때 까지
			while (arr[i] < arr[pivot]) {
				i++;
				if (i >= end) { 
					i = pivot;
				}				
			}

			// <-- 방향으로 피벗값보다 작은 값 찾을때 까지
			while (arr[j] > arr[pivot]) {
				j--;
				if (j <= start) {
					j = pivot;
				}
			}

			if (i < j) {
				// 안 엇갈림 = 자리 바꿈
				temp = arr[j];
				arr[j] = arr[i];
				arr[i] = temp;
			} else {
				// 엇갈릴 때
				if (j != pivot) {
					temp = arr[j];
					arr[j] = arr[pivot];
					arr[pivot] = temp;
				}
			}

		} while (i < j); // 안엇갈렸다면 계속 반복
		
		//엇갈린 경우 바뀐 피벗값 기준으로 정렬 필요
		quickSort(arr, start, j - 1);
		quickSort(arr, j + 1, end);
	}
}
```

퀵 정렬은 pivot 값에 따라 비균등하게 리스트가 나뉘게 된다.  
딱 반절로 나뉘게 되는 경우가 최선의 경우인데 이 때에는 **O(NlogN)**의 시간복잡도를 가진다.  
반면, 1, n-1 개 이렇게 나뉘는 경우에는 한 쪽으로 치우치면서 **O(N^2)**의 시간복잡도를 가지게 된다.  

<br/>

<br/>

## 병합 정렬 (Merge Sort)

병합 정렬은 그림으로 이해해버리자  

![1](https://user-images.githubusercontent.com/45073750/102913277-e5389680-44c1-11eb-92b5-025d99569ba4.png)

```java
public class Main {

    public static int[] src;
    public static int[] tmp;
    public static void main(String[] args) {
        src = new int[]{1, 9, 8, 5, 4, 2, 3, 7, 6};
        tmp = new int[src.length];
        printArray(src);
        mergeSort(0, src.length-1);
        printArray(src);
    }

    public static void mergeSort(int start, int end) {
        if (start<end) {
            int mid = (start+end) / 2;
            mergeSort(start, mid);
            mergeSort(mid+1, end);

            int p = start;
            int q = mid + 1;
            int idx = p;

            while (p<=mid || q<=end) {
                if (q>end || (p<=mid && src[p]<src[q])) {
                    tmp[idx++] = src[p++];
                } else {
                    tmp[idx++] = src[q++];
                }
            }

            for (int i=start;i<=end;i++) {
                src[i]=tmp[i];
            }
        }
    }

    public static void printArray(int[] a) {
        for (int i=0;i<a.length;i++)
            System.out.print(a[i]+" ");
        System.out.println();
    }
}
```

분할하는 과정에서 logN 만큼이 걸려서 총 시간복잡도는 **O(NlogN)** 이다.  
퀵소트와는 달리 pivot 설정하는 것이 없기 때문에 성능이 항상 같고 시간복잡도도 일정하다.  

하지만, 단점은 **추가적인 메모리가 필요** 하다는 점이다. 병합정렬은 임시배열에 원본을 계속해서 옮겨주면서 정렬하는 방법이기 때문이다. 만약 추가적인 메모리를 할당할 수 없는 경우에는 퀵 정렬을 사용해야 할 것이다. 메모리 할당이 가능한 일반적인 경우에는 데이터 최악의 상황이 있는 퀵 정렬보다는 병합정렬을 많이 사용한다.  

<br/>

<br/>

## 힙 정렬 (Heap Sort)

최대 힙 트리나 최소 힙 트리를 구성해서 정렬하는 것을 말한다.  

> 힙이란 최댓값 및 최솟값을 찾아내는 연산을 빠르게 하기 위해 고안된 완전 이진 트리

![2](https://user-images.githubusercontent.com/45073750/103008561-aa456a00-4578-11eb-9fcf-110f2c97081b.png)

```java
public class HeapSort {

	public static void main(String[] args) {

		// 정렬 되지 않은 배열
        int[] arr = {5, 8, 4, 7, 10, 9, 2, 1, 6, 3};
        
		/* 
		 * < maxHeap 만들기 >
		 * - 부모노드의 값이 자식노드의 값들보다 큰 형태
		 * - i의 초기값은 배열의 제일 끝 자식노드의 부모노드부터 시작한다.
		 */
		for(int i=arr.length/2-1;i>=0;i--){
			heapify(arr, arr.length, i);
		}
		
		// 정렬하기
		for(int i=arr.length-1;i>=0;i--){			
			swap(arr, i, 0); // 최상단 노드와 최하단 노드 값을 교환한다.
			heapify(arr, i-1, 0); // 루트노드를 기준으로 최대힙을 만든다.
		}
        
        // 출력
        for(int i=0;i<arr.length;i++){
        	System.out.print(arr[i]+" ");
        }
	}
	
	public static void heapify(int[] arr, int size, int pNode){
		
		int parent = pNode; // 부모노드
		int lNode = pNode * 2 + 1; // 왼쪽 자식노드
		int rNode = pNode * 2 + 2; // 오른쪽 자식노드
		
		// size > lNode => 인덱스 범위를 넘어서는지 확인하기 위함
		if(size > lNode && arr[parent] < arr[lNode]){
			parent = lNode;
		}
		
		if(size > rNode && arr[parent] < arr[rNode]){
			parent = rNode;
		}
		
		// parent에 저장된 값은 자식노드 중 큰 값이 있다면 큰 값의 인덱스 값이 남아있을 것이다.
		// 초기에 설정한 부모노드의 인덱스와 다르다면 교환을 해준다.
		if(parent != pNode){
			swap(arr, pNode, parent);
			
			/* 
			 * 노드와 자리를 바꾸면 최대힙 기준에 맞지 않을 수 있기 때문에 
			 * 바뀐 자식노드 아래 최대힙으로 맞춰주기 위함
			 */
			heapify(arr, size, parent); 
		}
	}
	
	public static void swap(int[] arr, int i, int j){
		
		int tmp = arr[j];
		arr[j] = arr[i];
		arr[i] = tmp;
	}
}
```

힙 정렬은 추가적인 메모리를 필요로 하지 않으면서 항상 **O(NlogN)**이라는 시간복잡도를 가진다.  
이상적인 경우에 퀵 정렬과 비교했을 때 시간복잡도는 같지만 실제 시간을 측정해보면 퀵 정렬보다 느리게 나온다. 즉, 데이터의 상태에 따라서 다른 정렬법들에 비해서 느린 편이다.  
또한, 안정성(Stable)을 보장받지 못한다는 단점이 있다.  

> 안정성(Stable) : 정렬되지 않은 상태에서 같은 키 값을 가진 원소의 순서가 정렬 후에도 유지되는 것

<br/>

<br/>

***

### Reference  

선택정렬 이미지 : https://devuna.tistory.com/28

삽입정렬 이미지 : https://dpdpwl.tistory.com/18  

버블정렬 이미지 : https://yeon6852.tistory.com/49  

병합정렬 이미지 및 코드: https://yunmap.tistory.com/entry/%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-Java%EB%A1%9C-%EA%B5%AC%ED%98%84%ED%95%98%EB%8A%94-%EC%89%AC%EC%9A%B4-Merge-Sort-%EB%B3%91%ED%95%A9-%EC%A0%95%EB%A0%AC-%ED%95%A9%EB%B3%91-%EC%A0%95%EB%A0%AC  

퀵정렬 이미지 및 코드 : https://dabo-dev.tistory.com/16  

힙정렬 이미지 : https://www.pinterest.co.kr/pin/463378249163376581/  

힙정렬 코드 : https://hmkim829.tistory.com/9



