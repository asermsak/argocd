# Deploy Web Server
Kubenetes ค่อนข้างซับซ้อนอ่านเข้าใจยาก ใช้เวลาเรียนนาน การเรียนจากแบบฝึกหัด ได้ใช้งานจริง จะทำให้เรียนรู้และเข้าใจชัดเจนและรวดเร็วขึ้น เพื่อให้เรียบง่ายจะแสดงการ deploy nginx ขึ้น Kubenetes  เนื้อหานี้เหมาะกับ Developer ที่ใช้ Docker มาบ้างแล้ว เพื่อที่จะได้นำโปรแกรมที่สร้างขึ้น K8s ได้ด้วยตัวเอง การทำ Auto deploy สำหรับ DevOps จะได้ทำในหัวข้อต่อๆไป จะได้เรียนหัวข้อต่างๆดังนี้

# สิ่งที่จะได้เรียนรู้
- Namespace ใช้แบ่ง cluster ออกจากกันแบบเสมือน ไม่ต้องแยกเครื่อง เวลาทำงานคนละ Namespace จะไม่เห็นและยุ่งกัน กันคนใช้เผลอไปทำงานคนอื่นพังได้ เวลาจะลบสิ่งที่ทำไว้ก็แค่ลบ Namespace ออก
- Deployment เป็นการ nginx container (Pod)ขึ้น Kubenetes ตาม spec ที่กำหนดไว้ เราจะทำการ mount Volume ตรงส่วนนี้
- Service จะเป็นการเปิดพอร์ตการติดต่อกับ pod ตัว Ingress จะผ่านช่องทางนี้
- Ingress ทำหน้าที่เป็น Reverse Proxy ดูจากที่ทำการ Request เข้ามาและส่งต่อให้ Service ตาม Rule ที่กำหนด
- hostPath เพื่อ mount volume กับไฟล์หรือโฟลเดอร์ที่อยู่บนเครื่อง(node) จะเหมาะกับแบบ Single Node เช่นระบบ Test และ Development สำหรับแบบ Cluster แนะนำให้ใช้ NFS หรือ Longhorn
- ConfigMap ไว้อ้างถึงค่าคอนฟิกระบบที่ไม่เป็นความลับ ในตัวอย่างจะใช้การ Mount กับ Volume
- Secret คล้าย ConfigMap แนะเหมาะสำหรับกับการเก็บค่าที่ไม่อยากให้คนเห็นได้ง่ายเหมือน config เช่น token, รหัสผ่าน, connection string ฯลฯ


## คำสั่งที่ใช้
คำสั่งส่วนใหญ่ผมจะ redirect output ไปเป็นไฟล์

``` bash
# คำสั้ง create เพื่อสรัาง Namespace, Deployment, Service และ Ingress
kubectl create ns my-web --dry-run=client -o yaml
kubectl create deployment myweb --image=nginx -n my-web --dry-run=client -o yaml
kubectl create service clusterip myweb --tcp=80:80 -n my-web --dry-run=client -o yaml 
kubectl create ingress myweb --rule="myweb.demo.local/*=myweb:80" -n my-web --dry-run=client -o yaml 
# ConfigMap และ Secret
kubectl create configmap myweb-configmap --from-file=./config/config.html --from-file=config1.html=./config/config.txt -n my-web --dry-run=client -o yaml
kubectl create secret generic myweb-secret --from-literal=username=sroemsak --from-literal=password=1234567 -n my-web --dry-run=client -o yaml
# ตัวอย่างการเข้ารหัสด้วย base64
echo -n "<password>" | base64
echo "<username>" | base64 -d
kubectl get pod -n myweb
kubectl exec -it <pod name> -n my-web -- /bin/bash
ls /usr/share/nginx/secret
# ลบทิ้งทั้งหมด
kubectl delete ns my-web
```

## Mount
ตัวอย่างโค้ดการ Mount ดูใน [deploy.yaml](./deploy.yaml) 
``` yaml
...
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:
        - name: hostpath-volume
          mountPath: /usr/share/nginx/html
        - name: configmap-volume
          mountPath: /usr/share/nginx/html/configmap
        - name: secret-volume
          mountPath: /usr/share/nginx/secret
      volumes:
      - name: hostpath-volume
        hostPath:
          path: /home/sroemsak/k8s/myweb/html
          type: Directory
      - name: configmap-volume
        configMap:
          name: myweb-configmap
      - name: secret-volume
        secret:
          secretName: myweb-secret
...
```
ถ้าเปิดเวปแล้วไม่เห็นให้ลองวิเคราะห์ดูจาก Traefik dashboard หาชื่อ pod แล้ว forward port แล้วไปที่ลิงค์

```
kubectl -n kube-system get pods
kubectl -n kube-system port-forward traefik-64b96ccbcd-t9k54 9000:9000 --address 0.0.0.0
```


